---
title: 协程约定与原理
description: Prism 协程编程规范、C++23 协程原理、使用示例、最佳实践与排障指南
layer: dev
---

# 协程约定

Prism 采用 **C++23 纯协程架构**，所有异步操作均基于 Boost.Asio 协程实现。本文档定义协程编程的约定、原理、示例和排障指南。

## 基本约定

### 异步操作返回类型

所有异步操作返回 `net::awaitable<T>`：

```cpp
namespace net = boost::asio;

// 异步连接
auto connect(const endpoint& ep) -> net::awaitable<connection_ptr>;

// 异步读取
auto read_some(buffer_span buffer) -> net::awaitable<size_t>;

// 异步写入
auto write_some(const_buffer_span data) -> net::awaitable<size_t>;
```

### 顺序异步操作

使用 `co_await` 执行顺序异步操作：

```cpp
auto handle_session(tcp::socket socket) -> net::awaitable<void> {
    // 顺序执行异步操作
    auto data = co_await read_data(socket);
    auto result = co_await process_data(data);
    co_await write_response(socket, result);
}
```

### 启动独立协程

使用 `net::co_spawn` 启动独立协程：

```cpp
// 启动独立协程处理连接
net::co_spawn(
    io_context,
    handle_connection(std::move(socket)),
    net::detached
);
```

## 协程纯度要求

Prism 采用**纯协程架构**，禁止在协程中使用任何阻塞操作。

### 禁止事项表

| 禁止操作 | 替代方案 |
|----------|----------|
| `std::mutex` / `std::lock_guard` | `std::atomic`、`strand`、`concurrent_channel` |
| `std::this_thread::sleep_for()` | `net::steady_timer::async_wait()` |
| 阻塞 socket read/write | `async_read_some` / `async_write_some` |
| `::getaddrinfo()` 同步 DNS | `resolver.async_resolve()` |
| `std::future::get()` / `wait()` | `co_await` 异步结果 |
| `while (!flag) {}` 忙等待 | `co_await` 异步等待 + 通知机制 |

### 详细说明

#### 1. 禁止互斥锁

协程中禁止使用 `std::mutex` 或 `std::lock_guard`，会导致线程阻塞：

```cpp
// 禁止
net::awaitable<void> bad_example() {
    std::lock_guard<std::mutex> lock(mutex_);  // 阻塞！
    co_await do_something();
}

// 推荐：使用 strand 或原子操作
net::awaitable<void> good_example() {
    // 使用 strand 保证串行执行
    co_await net::post(strand_, net::use_awaitable);
    do_something();
}
```

#### 2. 禁止阻塞睡眠

协程中禁止使用同步睡眠：

```cpp
// 禁止
net::awaitable<void> bad_sleep() {
    std::this_thread::sleep_for(std::chrono::seconds(1));  // 阻塞！
    co_return;
}

// 推荐：使用异步定时器
net::awaitable<void> good_sleep() {
    net::steady_timer timer(co_await net::this_coro::executor);
    timer.expires_after(std::chrono::seconds(1));
    co_await timer.async_wait(net::use_awaitable);
}
```

#### 3. 禁止阻塞 I/O

所有 socket 操作必须使用异步版本：

```cpp
// 禁止
void bad_io(tcp::socket& socket) {
    char buffer[1024];
    size_t n = socket.read_some(net::buffer(buffer));  // 阻塞！
}

// 推荐：使用异步读取
net::awaitable<void> good_io(tcp::socket& socket) {
    char buffer[1024];
    size_t n = co_await socket.async_read_some(
        net::buffer(buffer),
        net::use_awaitable
    );
}
```

#### 4. 禁止同步 DNS

使用异步 DNS 解析：

```cpp
// 禁止
net::awaitable<void> bad_dns() {
    auto results = net::ip::tcp::resolver::query("example.com", "80");
    // 同步解析会阻塞
}

// 推荐：异步解析
net::awaitable<void> good_dns() {
    net::ip::tcp::resolver resolver(co_await net::this_coro::executor);
    auto results = co_await resolver.async_resolve(
        "example.com", "80",
        net::use_awaitable
    );
}
```

#### 5. 禁止阻塞 Future

不要在协程中阻塞等待 `std::future`：

```cpp
// 禁止
net::awaitable<void> bad_future() {
    auto future = std::async(std::launch::async, heavy_work);
    auto result = future.get();  // 阻塞！
    co_return;
}

// 推荐：使用协程组合或 concurrent_channel
```

#### 6. 禁止忙等待

不要使用自旋锁或忙等待：

```cpp
// 禁止
net::awaitable<void> bad_spin() {
    while (!flag_) {
        // 浪费 CPU，阻塞协程调度
    }
    co_return;
}

// 推荐：使用异步通知机制
net::awaitable<void> good_notify() {
    co_await condition_.async_wait(net::use_awaitable);
}
```

## 协程启动模式

### 按 `self` 捕获

`co_spawn` 的 lambda 必须按值捕获 `self`（shared_ptr）保持对象存活：

```cpp
void start_session() {
    auto self = shared_from_this();
    net::co_spawn(
        io_context_,
        [self]() -> net::awaitable<void> {
            co_await self->handle_session();
        },
        net::detached
    );
}
```

### 使用 `net::detached`

对于不需要等待结果的协程，使用 `net::detached`：

```cpp
net::co_spawn(
    io_context,
    background_task(),
    net::detached
);
```

---

## 协程原理

### 协程的本质

协程是一种可以挂起和恢复执行的函数。与普通函数的区别：

```
普通函数:
调用 → 执行 → 返回 → 结束（一次性）

协程:
调用 → 执行 → 挂起 → 恢复 → 挂起 → ... → 返回 → 结束（可暂停）
```

| 特性 | 说明 |
|------|------|
| 挂起 | 协程暂停执行，保存当前状态 |
| 恢复 | 协程继续执行，恢复之前的状态 |
| 状态保存 | 协程帧（Coroutine Frame）存储局部变量和执行位置 |
| 用户态切换 | 协程切换不涉及内核，开销极小 |

### C++ 协程关键字

| 关键字 | 作用 | 示例 |
|--------|------|------|
| `co_await` | 挂起协程，等待异步操作 | `auto data = co_await socket.read();` |
| `co_yield` | 生成值并挂起（用于生成器） | `co_yield value;` |
| `co_return` | 从协程返回值 | `co_return result;` |

### 协程 vs 传统异步模型

| 模型 | 代码风格 | 内存开销 | 切换开销 | 并发能力 |
|------|----------|----------|----------|----------|
| 回调 | 嵌套回调，回调地狱 | 几乎无 | 无切换 | 百万级 |
| 线程池 | 同步风格，需要锁 | 每线程 ~8MB 栈 | 内核态 ~1-10us | 数千 |
| Future/Promise | 链式调用，较复杂 | 每任务 ~几百B | 无切换 | 数万 |
| 协程 | 线性代码，接近同步 | 每协程 ~256B-2KB | 用户态 ~10ns | 百万级 |

### Boost.Asio 协程操作

| 操作 | 用法 |
|------|------|
| 异步读取 | `co_await socket.async_read_some(buf, net::use_awaitable)` |
| 异步写入 | `co_await socket.async_write_some(buf, net::use_awaitable)` |
| 异步等待 | `co_await timer.async_wait(net::use_awaitable)` |
| 异步接受 | `co_await acceptor.async_accept(net::use_awaitable)` |
| 异步解析 | `co_await resolver.async_resolve(host, port, net::use_awaitable)` |
| 启动协程 | `net::co_spawn(io_context, task, net::detached)` |

### 协程底层机制

#### 协程帧（Coroutine Frame）

每个协程在挂起时需要一个协程帧保存状态：

```
+----------------------------------+
|       Coroutine Frame            |
+----------------------------------+
| Promise 对象                     |  <- 控制协程行为
| 参数副本                          |  <- 函数参数
| 局部变量                          |  <- 协程内的变量
| 挂起点索引                        |  <- 记录执行位置
| 对齐填充                          |  <- 内存对齐
+----------------------------------+
```

帧大小取决于局部变量数量、挂起点数量和 Promise 类型。典型代理连接协程帧约 256B-2KB。

#### Promise Type

每个协程类型关联一个 Promise 类型，控制协程行为：

```cpp
template<typename T>
struct task_promise {
    T result_;
    std::coroutine_handle<> continuation_;

    Task<T> get_return_object() {
        return Task<T>{std::coroutine_handle<task_promise>::from_promise(*this)};
    }

    std::suspend_always initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept { return {}; }
    void return_value(T value) { result_ = std::move(value); }
    void unhandled_exception() { std::terminate(); }
};
```

#### Awaitable 接口

`co_await` 表达式要求对象实现 Awaitable 接口：

```cpp
struct Awaitable {
    bool await_ready() { return false; }

    void await_suspend(std::coroutine_handle<> h) {
        continuation_ = h;
        start_async_operation();
    }

    auto await_resume() { return result_; }
};
```

#### 协程生命周期

```
调用协程函数
    │
    ▼
创建协程帧 + Promise
    │
    ▼
initial_suspend() ──── 挂起（等待被启动）
    │
    ▼
net::co_spawn() ────── 安排到执行器
    │
    ▼
恢复执行
    │
    ▼
遇到 co_await
    │
    ▼
await_ready() ──────── 检查是否需要挂起
    │ (false)
    ▼
await_suspend() ────── 挂起，启动异步操作
    │
    ▼
异步操作完成
    │
    ▼
恢复协程
    │
    ▼
await_resume() ─────── 获取结果
    │
    ▼
继续执行 ...
    │
    ▼
co_return
    │
    ▼
return_value() ─────── 保存返回值
    │
    ▼
final_suspend() ────── 最终挂起
    │
    ▼
协程帧销毁
```

### 协程调度模型

#### io_context 和 strand

```
+------------------------+
|     io_context         |  <- 每个 worker 一个
+------------------------+
          │
          ▼
+------------------------+
|       strand           |  <- 序列化执行
+------------------------+
          │
    +-----+-----+
    │           │
+-------+   +-------+
|协程 A |   |协程 B |  <- 同 strand 内顺序执行
+-------+   +-------+
```

strand 确保同一组协程顺序执行，无需锁。

#### 协程 spawn 和执行

```cpp
void launch_session(Socket socket, Worker& worker) {
    auto executor = worker.executor();
    net::co_spawn(
        executor,
        session_coroutine(std::move(socket)),
        net::detached
    );
}

net::awaitable<void> session_coroutine(Socket socket) {
    auto executor = co_await net::this_coro::executor;
    // 协程内所有操作使用同一执行器
}
```

---

## 使用示例

### 基础协程示例

```cpp
net::awaitable<std::string> fetch_url(std::string url) {
    auto executor = co_await net::this_coro::executor;

    auto resolver = net::ip::tcp::resolver(executor);
    auto endpoints = co_await resolver.async_resolve(url, "80", net::use_awaitable);

    auto socket = net::ip::tcp::socket(executor);
    co_await net::async_connect(socket, endpoints, net::use_awaitable);

    std::string request = "GET / HTTP/1.1\r\nHost: " + url + "\r\n\r\n";
    co_await net::async_write(socket, net::buffer(request), net::use_awaitable);

    std::array<char, 4096> buffer;
    auto n = co_await socket.async_read_some(net::buffer(buffer), net::use_awaitable);

    co_return std::string(buffer.data(), n);
}
```

### 双向隧道转发

```cpp
net::awaitable<void> tunnel(Socket client, Socket upstream) {
    auto executor = co_await net::this_coro::executor;

    auto up_task = [](Socket client, Socket upstream) -> net::awaitable<void> {
        std::array<char, 8192> buffer;
        while (true) {
            auto n = co_await client.async_read_some(buffer, net::use_awaitable);
            if (n == 0) break;
            co_await net::async_write(upstream, net::buffer(buffer, n), net::use_awaitable);
        }
    };

    auto down_task = [](Socket upstream, Socket client) -> net::awaitable<void> {
        std::array<char, 8192> buffer;
        while (true) {
            auto n = co_await upstream.async_read_some(buffer, net::use_awaitable);
            if (n == 0) break;
            co_await net::async_write(client, net::buffer(buffer, n), net::use_awaitable);
        }
    };

    co_await net::experimental::parallel_group(
        net::co_spawn(executor, up_task(client, upstream), net::use_awaitable),
        net::co_spawn(executor, down_task(upstream, client), net::use_awaitable)
    );
}
```

### 协程组合

```cpp
net::awaitable<std::vector<std::string>> fetch_all(std::vector<std::string> urls) {
    auto executor = co_await net::this_coro::executor;

    std::vector<net::awaitable<std::string>> tasks;
    for (auto& url : urls) {
        tasks.push_back(net::co_spawn(executor, fetch_url(url), net::use_awaitable));
    }

    auto results = co_await net::experimental::parallel_group(tasks);
    co_return results;
}
```

### 协程错误处理

```cpp
net::awaitable<void> safe_operation() {
    try {
        auto result = co_await risky_operation();
    } catch (const std::exception& e) {
        log_error("Operation failed: {}", e.what());
        co_return;
    }
}
```

### 协程超时处理

```cpp
net::awaitable<std::optional<std::string>> fetch_with_timeout(std::string url, int timeout_ms) {
    auto executor = co_await net::this_coro::executor;

    auto timer = net::steady_timer(executor);
    timer.expires_after(std::chrono::milliseconds(timeout_ms));

    auto task = net::co_spawn(executor, fetch_url(url), net::use_awaitable);
    auto timeout_task = timer.async_wait(net::use_awaitable);

    auto [order, result, timeout_result] = co_await net::experimental::parallel_group(
        task, timeout_task
    );

    if (order == 0) {
        co_return result;
    } else {
        co_return std::nullopt;
    }
}
```

---

## 最佳实践

### 协程命名规范

| 类型 | 前缀 | 示例 |
|------|------|------|
| 返回值协程 | Task/awaitable | `Task<Data> fetchData()` |
| 无返回协程 | awaitable<void> | `awaitable<void> process()` |
| 生成器协程 | Generator | `Generator<int> numbers()` |

### 资源管理

```cpp
net::awaitable<void> safe_resource_use() {
    ResourceGuard guard(resource);  // RAII 自动管理
    co_await use_resource();
    // 协程结束时 guard 自动清理，即使异常也会清理
}
```

### 避免 co_await 后的迭代器失效

```cpp
// 错误：co_await 后迭代器可能失效
net::awaitable<void> bad_iterator() {
    auto& container = get_container();
    auto it = container.begin();
    co_await some_operation();  // 其他协程可能修改 container
    use(*it);  // it 可能已失效！
}

// 正确：co_await 后重新获取
net::awaitable<void> good_iterator() {
    auto& container = get_container();
    co_await some_operation();
    auto it = container.begin();  // 重新获取
    use(*it);
}
```

### 协程中的 self 持有

`co_spawn` 的 lambda 必须按值捕获 `self`：

```cpp
net::awaitable<void> spawn_subtask(std::shared_ptr<Session> self) {
    net::co_spawn(
        co_await net::this_coro::executor,
        [self]() -> net::awaitable<void> {  // 按值捕获
            co_await self->subtask();
        },
        net::detached
    );
}
```

### PMR 内存池与协程

Prism 使用 PMR 管理协程内存，协程帧优先从线程本地池分配，避免锁竞争：

```
+------------------------+
|    线程本地池 (TL)      |  <- 协程帧优先从这里分配
+------------------------+
          │ (耗尽时)
          ▼
+------------------------+
|    全局池 (Global)      |  <- 备用池，有锁竞争
+------------------------+
          │ (超大对象)
          ▼
+------------------------+
|    系统堆 (Heap)        |  <- 最后手段，性能最低
+------------------------+
```

```cpp
net::awaitable<void> process_request(Socket socket) {
    memory::vector<uint8_t> buffer;  // 从帧池分配
    auto n = co_await socket.async_read_some(buffer, net::use_awaitable);
    // 帧结束时自动清理，无需手动释放
}
```

---

## 常见问题与排障

### Q: 协程为什么比线程更高效？

- 内存开销：协程帧 ~1KB，线程栈 ~8MB
- 切换开销：协程切换用户态 ~10ns，线程切换内核态 ~1us
- 调度开销：协程由用户控制，线程由内核调度

### Q: 协程可以嵌套吗？

可以。使用 `co_await` 等待子协程：

```cpp
net::awaitable<void> parent_coroutine() {
    auto result = co_await child_coroutine();
}

net::awaitable<int> child_coroutine() {
    co_await some_work();
    co_return 42;
}
```

### Q: 如何在协程中处理大量并发？

使用并行组和适当的并发限制：

```cpp
net::awaitable<void> process_many() {
    auto semaphore = std::make_shared<net::experimental::semaphore>(100);

    std::vector<net::awaitable<void>> tasks;
    for (int i = 0; i < 1000; ++i) {
        tasks.push_back([semaphore]() -> net::awaitable<void> {
            co_await semaphore->acquire();
            co_await process_one();
            semaphore->release();
        });
    }

    co_await net::experimental::parallel_group(tasks);
}
```

### Q: 协程中的异常如何处理？

- 异常会传播到 `unhandled_exception()`，默认调用 `std::terminate()`
- 可以在 Promise 中自定义异常处理
- 建议使用 try-catch 包裹可能失败的代码

### Q: 协程可以取消吗？

可以，通过取消信号：

```cpp
net::awaitable<void> cancellable_task(std::atomic<bool>& cancelled) {
    while (!cancelled) {
        co_await do_work();
    }
}
```

---

### 问题：协程挂起不恢复

**症状**: 协程启动后不继续执行

**排查步骤**:

1. 检查 `io_context.run()` 是否调用
2. 检查 `await_suspend` 是否保存了 handle，异步操作完成时是否调用 resume
3. 检查是否有死锁（协程等待永远不会发生的事件）

### 问题：协程崩溃

**症状**: 协程执行中段错误

**排查步骤**:

1. 检查协程帧是否被销毁 — co_await 返回后，之前的引用可能失效
2. 不要保存协程内的指针
3. 检查 RAII 对象是否正确管理资源
4. 使用调试器查看调用栈

### 问题：协程内存泄漏

**症状**: 内存持续增长

**排查步骤**:

1. 检查协程是否正确结束 — `final_suspend` 是否返回 `suspend_always`
2. 检查协程内的资源是否清理
3. 使用内存分析工具：`valgrind ./program`

### 问题：协程切换慢

**症状**: 协程恢复延迟高

**排查步骤**:

1. 检查 io_context 负载 — 是否有过多的任务排队或阻塞操作影响调度
2. 检查是否有锁竞争（strand 使用不当、共享资源竞争）
3. 增加 worker 数量

---

## 相关文档

- [[overview]] - 编码规范概览
- [[lifecycle]] - 生命周期安全指南
- [[dev/modules]] - 模块结构
- [[pmr-memory-pool]] - 协程内存管理
- [[dev/debugging/tcp|tcp]] - TCP 协议协程实践
