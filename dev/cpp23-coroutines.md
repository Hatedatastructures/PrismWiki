---
title: C++23 协程深度指南
source:
  - I:/code/Prism/include/prism/
  - I:/code/Prism/src/prism/
  - I:/code/Prism/CLAUDE.md
module: coroutine
type: concept
tags: [cpp, coroutine, async, networking, awaitable, boost-asio]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[modules/agent]]"
  - "[[pmr-memory-pool]]"
  - "[[dev/tcp]]"
  - "[[architecture]]"
---

# C++23 协程深度指南

C++23 协程是现代 C++ 异步编程的核心范式。Prism 采用纯协程架构，全链路异步实现高性能代理引擎。

## 概述

### 协程的本质

协程是一种可以挂起和恢复执行的函数。与普通函数的区别：

```
普通函数:
调用 → 执行 → 返回 → 结束（一次性）

协程:
调用 → 执行 → 挂起 → 恢复 → 挂起 → ... → 返回 → 结束（可暂停）
```

协程的关键特性：

| 特性 | 说明 |
|------|------|
| 挂起 | 协程暂停执行，保存当前状态 |
| 恢复 | 协程继续执行，恢复之前的状态 |
| 状态保存 | 协程帧（Coroutine Frame）存储局部变量和执行位置 |
| 用户态切换 | 协程切换不涉及内核，开销极小 |

### C++ 协程关键字

C++20 引入三个协程关键字：

| 关键字 | 作用 | 示例 |
|--------|------|------|
| `co_await` | 挂起协程，等待异步操作 | `auto data = co_await socket.read();` |
| `co_yield` | 生成值并挂起（用于生成器） | `co_yield value;` |
| `co_return` | 从协程返回值 | `co_return result;` |

使用这些关键字的函数自动成为协程：

```cpp
// 普通函数
std::string fetchData(std::string url);

// 协程函数（包含 co_await）
Task<std::string> fetchData(std::string url) {
    auto response = co_await httpGet(url);
    co_return response.body;
}
```

### 协程 vs 传统异步模型

| 模型 | 代码风格 | 内存开销 | 切换开销 | 并发能力 |
|------|----------|----------|----------|----------|
| 回调 | 嵌套回调，回调地狱 | 几乎无 | 无切换 | 百万级 |
| 线程池 | 同步风格，需要锁 | 每线程 ~8MB 栈 | 内核态 ~1-10us | 数千 |
| Future/Promise | 链式调用，较复杂 | 每任务 ~几百B | 无切换 | 数万 |
| 协程 | 线性代码，接近同步 | 每协程 ~256B-2KB | 用户态 ~10ns | 百万级 |

协程在代码可读性和并发能力上实现最佳平衡。

### Boost.Asio 协程支持

Prism 使用 Boost.Asio 作为异步 I/O 框架，它原生支持 C++ 协程：

```cpp
namespace net = boost::asio;

// Asio 的 awaitable 类型
net::awaitable<T> task();

// 使用 co_await
auto data = co_await socket.async_read(buffer, net::use_awaitable);
co_await socket.async_write(data, net::use_awaitable);
co_await timer.async_wait(net::use_awaitable);
```

核心 Asio 协程操作：

| 操作 | 用法 |
|------|------|
| 异步读取 | `co_await socket.async_read_some(buf, net::use_awaitable)` |
| 异步写入 | `co_await socket.async_write_some(buf, net::use_awaitable)` |
| 异步等待 | `co_await timer.async_wait(net::use_awaitable)` |
| 异步接受 | `co_await acceptor.async_accept(net::use_awaitable)` |
| 异步解析 | `co_await resolver.async_resolve(host, port, net::use_awaitable)` |
| 启动协程 | `net::co_spawn(io_context, task, net::detached)` |

---

## 详解

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

帧大小取决于：
- 局部变量数量和大小
- 挂起点数量
- Promise 类型

典型代理连接协程帧约 256B-2KB。

#### Promise Type

每个协程类型关联一个 Promise 类型，控制协程行为：

```cpp
// Task<T> 的 Promise（简化）
template<typename T>
struct task_promise {
    T result_;
    std::coroutine_handle<> continuation_;
    
    // 获取返回对象
    Task<T> get_return_object() {
        return Task<T>{std::coroutine_handle<task_promise>::from_promise(*this)};
    }
    
    // 初始挂起（不立即执行）
    std::suspend_always initial_suspend() { return {}; }
    
    // 最终挂起（等待被销毁）
    std::suspend_always final_suspend() noexcept { return {}; }
    
    // 处理返回值
    void return_value(T value) { result_ = std::move(value); }
    
    // 处理异常
    void unhandled_exception() { std::terminate(); }
};
```

#### Awaitable 接口

`co_await` 表达式要求对象实现 Awaitable 接口：

```cpp
// Awaitable 接口（简化）
struct Awaitable {
    // 是否立即挂起
    bool await_ready() { return false; }
    
    // 挂起时执行（保存恢复点）
    void await_suspend(std::coroutine_handle<> h) {
        // 保存 handle 以便恢复
        continuation_ = h;
        // 启动异步操作
        start_async_operation();
    }
    
    // 恢复时执行（获取结果）
    auto await_resume() {
        return result_;
    }
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

---

### Prism 协程应用场景

#### 每连接一协程模型

Prism 为每个客户端连接创建独立协程：

```
客户端连接 ──── spawn 协程
                     │
                     ▼
              +------------------+
              |  连接协程生命周期 |
              +------------------+
              │ co_await accept  │ 接受连接
              │ co_await probe   │ 协议探测
              │ co_await handshake│ TLS 握手
              │ co_await parse   │ 解析协议
              │ co_await resolve │ DNS 解析
              │ co_await connect │ 建立上游
              │ 循环转发          │ 双向隧道
              │ co_await close   │ 关闭连接
              +------------------+
                     │
                     ▼
              协程销毁，资源归还
```

优势：
- 每个连接独立状态，无共享竞争
- 协程内代码线性，易理解易维护
- 自动资源管理（协程结束自动清理）

#### 异步 DNS 解析

DNS 解析是高频操作，使用协程实现无阻塞：

```cpp
/**
 * @brief 异步 DNS 解析
 * @param hostname 目标主机名
 * @return 解析得到的 IP 地址
 */
net::awaitable<IPAddress> resolveDNS(std::string hostname) {
    auto resolver = net::ip::tcp::resolver(co_await net::this_coro::executor);
    
    // 异步解析，不阻塞线程
    auto results = co_await resolver.async_resolve(hostname, "443", net::use_awaitable);
    
    // 返回第一个结果
    for (auto& endpoint : results) {
        co_return endpoint.address();
    }
    
    // 无结果时返回错误
    throw std::runtime_error("DNS resolution failed");
}
```

#### 连接池管理

连接池的获取和复用使用协程：

```cpp
/**
 * @brief 从连接池获取连接
 * @param pool 连接池引用
 * @return 可用连接
 */
net::awaitable<Connection> getConnection(ConnectionPool& pool) {
    // 尝试立即获取
    auto conn = pool.tryAcquire();
    if (conn) {
        co_return conn;
    }
    
    // 无可用连接，等待
    auto timer = net::steady_timer(co_await net::this_coro::executor);
    timer.expires_after(std::chrono::milliseconds(100));
    
    while (!conn) {
        co_await timer.async_wait(net::use_awaitable);
        conn = pool.tryAcquire();
        timer.expires_after(std::chrono::milliseconds(100));
    }
    
    co_return conn;
}
```

#### 双向隧道转发

代理的核心是双向数据转发：

```cpp
/**
 * @brief 双向隧道转发
 * @param client 客户端连接
 * @param upstream 上游连接
 */
net::awaitable<void> tunnel(Socket client, Socket upstream) {
    // 使用 parallel_group 同时处理双向
    auto executor = co_await net::this_coro::executor;
    
    // 上行：client → upstream
    auto up_task = [](Socket client, Socket upstream) -> net::awaitable<void> {
        std::array<char, 8192> buffer;
        while (true) {
            auto n = co_await client.async_read_some(buffer, net::use_awaitable);
            if (n == 0) break;
            co_await net::async_write(upstream, net::buffer(buffer, n), net::use_awaitable);
        }
    };
    
    // 下行：upstream → client
    auto down_task = [](Socket upstream, Socket client) -> net::awaitable<void> {
        std::array<char, 8192> buffer;
        while (true) {
            auto n = co_await upstream.async_read_some(buffer, net::use_awaitable);
            if (n == 0) break;
            co_await net::async_write(client, net::buffer(buffer, n), net::use_awaitable);
        }
    };
    
    // 并行运行两个方向
    co_await net::experimental::parallel_group(
        net::co_spawn(executor, up_task(client, upstream), net::use_awaitable),
        net::co_spawn(executor, down_task(upstream, client), net::use_awaitable)
    );
}
```

---

### 协程纯度要求

Prism 采用纯协程架构，禁止在协程中使用任何阻塞操作：

#### 禁止的操作

| 禁止操作 | 替代方案 |
|----------|----------|
| `std::mutex` / `std::lock_guard` | `std::atomic`、strand、channel |
| `std::this_thread::sleep_for()` | `net::steady_timer::async_wait()` |
| 阻塞 socket read/write | `async_read_some`/`async_write_some` |
| `::getaddrinfo()` 同步 DNS | `resolver.async_resolve()` |
| `std::future::get()` / `wait()` | `co_await` 异步结果 |
| `while (!flag) {}` 忙等待 | `co_await` 异步等待 + 通知 |

#### 正确示例

```cpp
// 错误：使用 mutex
net::awaitable<void> bad_example() {
    std::lock_guard<std::mutex> lock(mutex_);  // 阻塞！
    // ...
}

// 正确：使用 strand
net::awaitable<void> good_example() {
    auto strand = net::make_strand(co_await net::this_coro::executor);
    // 在 strand 上执行，无需锁
    co_await net::co_spawn(strand, inner_task(), net::use_awaitable);
}

// 错误：使用 sleep
net::awaitable<void> bad_sleep() {
    std::this_thread::sleep_for(5s);  // 阻塞！
}

// 正确：使用 timer
net::awaitable<void> good_wait() {
    auto timer = net::steady_timer(co_await net::this_coro::executor);
    timer.expires_after(std::chrono::seconds(5));
    co_await timer.async_wait(net::use_awaitable);  // 异步等待
}
```

---

### 协程内存管理

#### PMR 内存池与协程

Prism 使用 PMR（Polymorphic Memory Resource）管理协程内存：

```cpp
// 协程中使用 PMR 容器
net::awaitable<void> process_request(Socket socket) {
    // 使用帧竞技场分配
    memory::vector<uint8_t> buffer;  // 从帧池分配
    
    auto n = co_await socket.async_read_some(buffer, net::use_awaitable);
    
    // 处理数据...
    
    // 帧结束时自动清理，无需手动释放
}
```

#### 协程帧分配策略

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

协程帧优先从线程本地池分配，避免锁竞争。

---

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
// 启动协程
void launch_session(Socket socket, Worker& worker) {
    auto executor = worker.executor();
    
    // co_spawn 创建并安排协程
    net::co_spawn(
        executor,
        session_coroutine(std::move(socket)),
        net::detached  // 不等待结果
    );
}

// 协程执行
net::awaitable<void> session_coroutine(Socket socket) {
    // 获取当前执行器
    auto executor = co_await net::this_coro::executor;
    
    // 协程内所有操作使用同一执行器
    // ...
}
```

---

## 使用示例

### 基础协程示例

```cpp
// 简单异步任务
net::awaitable<std::string> fetch_url(std::string url) {
    auto executor = co_await net::this_coro::executor;
    
    // 1. 解析地址
    auto resolver = net::ip::tcp::resolver(executor);
    auto endpoints = co_await resolver.async_resolve(url, "80", net::use_awaitable);
    
    // 2. 建立连接
    auto socket = net::ip::tcp::socket(executor);
    co_await net::async_connect(socket, endpoints, net::use_awaitable);
    
    // 3. 发送请求
    std::string request = "GET / HTTP/1.1\r\nHost: " + url + "\r\n\r\n";
    co_await net::async_write(socket, net::buffer(request), net::use_awaitable);
    
    // 4. 接收响应
    std::array<char, 4096> buffer;
    auto n = co_await socket.async_read_some(net::buffer(buffer), net::use_awaitable);
    
    // 5. 返回结果
    co_return std::string(buffer.data(), n);
}
```

### 协程组合

```cpp
// 并行执行多个协程
net::awaitable<std::vector<std::string>> fetch_all(std::vector<std::string> urls) {
    auto executor = co_await net::this_coro::executor;
    
    std::vector<net::awaitable<std::string>> tasks;
    for (auto& url : urls) {
        tasks.push_back(net::co_spawn(executor, fetch_url(url), net::use_awaitable));
    }
    
    // 等待所有完成
    auto results = co_await net::experimental::parallel_group(tasks);
    
    co_return results;
}
```

### 协程错误处理

```cpp
// 协程中的错误处理
net::awaitable<void> safe_operation() {
    try {
        auto result = co_await risky_operation();
        // 处理成功
    } catch (const std::exception& e) {
        // 协程中的异常处理
        log_error("Operation failed: {}", e.what());
        
        // 可以选择重试或返回错误
        co_return;  // 正常退出
    }
}

// 使用错误码而非异常
net::awaitable<fault::code> operation_with_error_code() {
    auto result = co_await async_operation();
    if (!result.success) {
        co_return result.error_code;
    }
    // 处理成功
    co_return fault::code::success;
}
```

### 协程超时处理

```cpp
// 带超时的操作
net::awaitable<std::optional<std::string>> fetch_with_timeout(std::string url, int timeout_ms) {
    auto executor = co_await net::this_coro::executor;
    
    // 创建定时器
    auto timer = net::steady_timer(executor);
    timer.expires_after(std::chrono::milliseconds(timeout_ms));
    
    // 启动任务
    auto task = net::co_spawn(executor, fetch_url(url), net::use_awaitable);
    auto timeout_task = timer.async_wait(net::use_awaitable);
    
    // 等任一完成
    auto [order, result, timeout_result] = co_await net::experimental::parallel_group(
        task, timeout_task
    );
    
    if (order == 0) {
        // 任务先完成
        co_return result;
    } else {
        // 超时
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
// RAII 在协程中的应用
net::awaitable<void> safe_resource_use() {
    // 使用 RAII 包装器
    ResourceGuard guard(resource);  // 自动管理
    
    // 协程内使用资源
    co_await use_resource();
    
    // 协程结束时 guard 自动清理
    // 即使异常也会清理
}
```

### 避免 co_await 后的失效

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
    
    // 重新获取迭代器
    auto it = container.begin();
    use(*it);
}
```

### 协程中的 self 持有

```cpp
// co_spawn 的 lambda 必须按值捕获 self
net::awaitable<void> spawn_subtask(std::shared_ptr<Session> self) {
    // 正确：self 按值捕获，保持引用
    net::co_spawn(
        co_await net::this_coro::executor,
        [self]() -> net::awaitable<void> {  // 按值捕获
            co_await self->subtask();
        },
        net::detached
    );
}
```

---

## 常见问题

### Q1: 协程为什么比线程更高效？

**A**: 
- 内存开销：协程帧 ~1KB，线程栈 ~8MB
- 切换开销：协程切换用户态 ~10ns，线程切换内核态 ~1us
- 调度开销：协程由用户控制，线程由内核调度

### Q2: 协程可以嵌套吗？

**A**: 可以。使用 `co_await` 等待子协程：

```cpp
net::awaitable<void> parent_coroutine() {
    // 调用子协程
    auto result = co_await child_coroutine();
    // 继续执行
}

net::awaitable<int> child_coroutine() {
    co_await some_work();
    co_return 42;
}
```

### Q3: 如何在协程中处理大量并发？

**A**: 使用并行组和适当的并发限制：

```cpp
net::awaitable<void> process_many() {
    // 使用 semaphore 限制并发
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

### Q4: 协程中的异常如何处理？

**A**: 
- 异常会传播到 `unhandled_exception()`，默认调用 `std::terminate()`
- 可以在 Promise 中自定义异常处理
- 建议使用 try-catch 包裹可能失败的代码

### Q5: 协程可以取消吗？

**A**: 可以，通过取消信号：

```cpp
net::awaitable<void> cancellable_task(std::atomic<bool>& cancelled) {
    while (!cancelled) {
        co_await do_work();
    }
    // 被取消时退出
}
```

---

## 排障指南

### 问题：协程挂起不恢复

**症状**: 协程启动后不继续执行

**排查步骤**:

1. 检查 io_context 是否运行
   ```cpp
   io_context.run();  // 必须调用
   ```

2. 检查异步操作是否正确启动
   - await_suspend 是否保存了 handle
   - 异步操作完成时是否调用 resume

3. 检查是否有死锁
   - 协程等待永远不会发生的事件

---

### 问题：协程崩溃

**症状**: 协程执行中段错误

**排查步骤**:

1. 检查协程帧是否被销毁
   - co_await 返回后，之前的引用可能失效
   - 不要保存协程内的指针

2. 检查资源生命周期
   - RAII 对象是否正确管理

3. 使用调试器查看调用栈

---

### 问题：协程内存泄漏

**症状**: 内存持续增长

**排查步骤**:

1. 检查协程是否正确结束
   - final_suspend 是否返回 suspend_always
   - 协程帧是否被销毁

2. 检查资源是否释放
   - 协程内的资源是否清理

3. 使用内存分析工具
   ```bash
   valgrind ./program
   ```

---

### 问题：协程切换慢

**症状**: 协程恢复延迟高

**排查步骤**:

1. 检查 io_context 负载
   - 是否有过多的任务排队
   - 是否有阻塞操作影响调度

2. 检查是否有锁竞争
   - strand 使用不当
   - 共享资源竞争

3. 增加 worker 数量

---

## 相关链接

- [[modules/agent]] — Agent 模块协程应用
- [[pmr-memory-pool]] — 协程内存管理
- [[dev/tcp]] — TCP 协议协程实践
- [[architecture]] — 整体架构设计