---
title: "C++23 协程"
category: "programming"
type: ref
module: ref
source: "C++20/C++23 Coroutines"
tags: [编程, c++, 协程, 异步, awaitable, promise_type, coroutine_handle, c++23]
created: 2026-05-15
updated: 2026-05-17
layer: ref
---

# C++23 协程

**类别**: 编程

## 概述

C++20 引入了协程（Coroutines）作为语言核心特性，C++23 在此基础上进行了扩展和优化。协程是一种可以被挂起和恢复的函数，允许以同步的风格编写异步代码，彻底告别回调地狱（Callback Hell）。Prism 代理服务器采用纯协程架构，所有异步操作均通过 `co_await` 实现，实现了高性能、可维护的异步 I/O 模型。

### 什么是协程

协程是能够暂停执行并稍后恢复的函数。与普通函数的"一次执行到底"不同，协程可以在执行过程中挂起（suspend），保存当前状态，然后在需要时恢复（resume）继续执行。这种特性使得协程非常适合表达异步操作：发起异步请求后挂起，等待操作完成后再恢复执行。

### 协程与传统异步模型的对比

**回调模式（Callback Pattern）**:
```cpp
// 回调地狱示例
socket.async_read(buffer, [](error_code ec, size_t bytes) {
    if (!ec) {
        socket.async_write(buffer, [](error_code ec, size_t bytes) {
            if (!ec) {
                socket.async_read(buffer, [](error_code ec, size_t bytes) {
                    // 嵌套越来越深...
                });
            }
        });
    }
});
```

**协程模式（Coroutine Pattern）**:
```cpp
// 协程版本：线性流程
auto bytes1 = co_await socket.async_read(buffer, net::use_awaitable);
auto bytes2 = co_await socket.async_write(buffer, net::use_awaitable);
auto bytes3 = co_await socket.async_read(buffer, net::use_awaitable);
```

协程将异步代码的编写体验提升到了接近同步代码的水平，同时保持了异步的高性能特性。

### C++20 协程的核心特性

C++20 协程是一个"栈无栈"（stackless）协程实现，其核心特性包括：

1. **语言级支持**: 三个新关键字 `co_await`、`co_return`、`co_yield`
2. **编译器魔法**: 编译器自动生成状态机，管理协程帧（coroutine frame）
3. **零成本抽象**: 无栈协程在挂起时不保存整个调用栈，只保存必要的状态
4. **可定制性**: 通过 `promise_type` 完全控制协程的行为

### C++23 对协程的增强

C++23 在协程方面增加了多项改进：

1. **`std::generator`**: 标准库生成器，简化 `co_yield` 的使用
2. **`std::ranges::elements_of`**: 支持生成器的嵌套 yield
3. **更完善的协程工具**: `std::coroutine_handle` 的更多操作
4. **改进的 awaiter 接口**: 更灵活的等待器设计
5. **`if consteval`**: 与协程配合使用的编译期分支

### 协程的生命周期

协程从创建到销毁经历以下阶段：

```
调用协程函数
     │
     ▼
分配协程帧（coroutine frame）
     │
     ▼
构造 promise_type 对象
     │
     ▼
初始化参数（复制到协程帧）
     │
     ▼
执行到第一个挂起点
     │
     ▼
返回给调用者（返回 awaitable）
     │
     ├──────────────┐
     ▼              │
挂起状态           │
     │              │
co_await 恢复 ─────┘
     │
     ▼
co_return 执行
     │
     ▼
销毁协程帧
```

## 原理详解

### 核心关键字

#### co_await

`co_await` 是协程的核心操作符，用于挂起当前协程并等待异步操作完成。其工作流程如下：

```cpp
auto result = co_await awaitable_expression;
```

当编译器遇到 `co_await expr` 时，会执行以下步骤：

1. **获取 awaiter**: 调用 `expr.operator co_await()` 或 `operator co_await(expr)`
2. **检查可挂起性**: 调用 `await_ready()`
   - 返回 `true`: 操作已完成，不挂起，直接返回结果
   - 返回 `false`: 操作未完成，准备挂起
3. **挂起协程**: 调用 `await_suspend(coroutine_handle)`
   - 返回 `void`: 挂起协程，稍后恢复
   - 返回 `bool`: `true` 挂起，`false` 不挂起
   - 返回 `coroutine_handle`: 挂起当前协程，恢复另一个协程
4. **恢复执行**: 当操作完成时，调用 `await_resume()` 获取结果

**自定义 Awaiter 示例**:

```cpp
struct delay_awaiter {
    net::steady_timer& timer;
    std::chrono::milliseconds duration;

    bool await_ready() const noexcept {
        return duration.count() <= 0;  // 零延迟不需要等待
    }

    void await_suspend(std::coroutine_handle<> handle) {
        timer.expires_after(duration);
        timer.async_wait([handle](std::error_code) mutable {
            handle.resume();  // 定时器到期后恢复协程
        });
    }

    void await_resume() const noexcept {
        // 无返回值
    }
};

// 使用方式
net::awaitable<void> delayed_operation() {
    net::steady_timer timer(co_await net::this_coro::executor);
    co_await delay_awaiter{timer, std::chrono::milliseconds(100)};
    // 100ms 后继续执行
}
```

#### co_return

`co_return` 用于从协程返回值，类似于普通函数的 `return`。协程只能使用 `co_return`，不能使用 `return`。

```cpp
net::awaitable<int> compute_value() {
    co_return 42;  // 返回值
}

net::awaitable<void> do_something() {
    co_return;  // 无返回值
}
```

`co_return` 执行时会：
1. 调用 `promise.return_value(value)` 或 `promise.return_void()`
2. 销毁局部变量
3. 挂起协程（处于最终挂起状态）

#### co_yield

`co_yield` 用于生成值，用于生成器协程。每次调用产生一个值并挂起协程。

```cpp
std::generator<int> fibonacci(int n) {
    int a = 0, b = 1;
    for (int i = 0; i < n; ++i) {
        co_yield a;  // 产出当前值
        auto tmp = a;
        a = b;
        b = tmp + b;
    }
}

// 使用方式
for (int value : fibonacci(10)) {
    std::cout << value << " ";
}
// 输出: 0 1 1 2 3 5 8 13 21 34
```

`co_yield value` 等价于 `co_await promise.yield_value(value)`。

### 协程帧与状态机

编译器为每个协程调用创建一个协程帧（coroutine frame），其中包含：

1. **Promise 对象**: `promise_type` 的实例
2. **协程参数**: 复制到堆上的参数
3. **局部变量**: 跨越挂起点的局部变量
4. **状态索引**: 状态机的当前状态
5. **保存的寄存器**: 挂起时的 CPU 上下文

**协程帧布局**:

```
+---------------------------+
| Promise Object            |
+---------------------------+
| Parameter 1               |
| Parameter 2               |
| ...                       |
+---------------------------+
| Local Variable 1          |
| Local Variable 2          |
| ...                       |
+---------------------------+
| State Index               |
+---------------------------+
| Saved Registers           |
+---------------------------+
```

编译器将协程转换为状态机，每个挂起点对应一个状态：

```cpp
// 原始协程
net::awaitable<void> example() {
    auto a = co_await op1();  // 状态 1
    auto b = co_await op2();  // 状态 2
    auto c = co_await op3();  // 状态 3
    co_return;
}

// 编译器生成的状态机伪代码
struct example_frame {
    int state = 0;
    decltype(op1())::value_type a;
    decltype(op2())::value_type b;
    decltype(op3())::value_type c;
    // ...
};

void example_resume(example_frame* frame) {
    switch (frame->state) {
        case 0: goto state_0;
        case 1: goto state_1;
        case 2: goto state_2;
        case 3: goto state_3;
    }
state_0:
    frame->state = 1;
    // co_await op1()
    // 如果挂起，返回；恢复时跳到 state_1
state_1:
    frame->state = 2;
    // co_await op2()
    // 如果挂起，返回；恢复时跳到 state_2
state_2:
    frame->state = 3;
    // co_await op3()
    // 如果挂起，返回；恢复时跳到 state_3
state_3:
    // co_return
    return;
}
```

### Promise Type

`promise_type` 是协程行为的定制点，控制协程的创建、挂起、返回等行为。每个协程类型都有一个关联的 `promise_type`。

**`promise_type` 的核心方法**:

| 方法 | 用途 | 说明 |
|------|------|------|
| `get_return_object()` | 创建返回对象 | 构造协程句柄并包装为返回类型 |
| `initial_suspend()` | 初始挂起点 | 返回 `suspend_always` 或 `suspend_never` |
| `final_suspend()` | 最终挂起点 | 返回 `suspend_always` 或 `suspend_never` |
| `return_value(value)` | 处理 `co_return value` | 存储返回值 |
| `return_void()` | 处理 `co_return;` | 无返回值时调用 |
| `yield_value(value)` | 处理 `co_yield value` | 存储产出的值 |
| `unhandled_exception()` | 处理异常 | 捕获并存储异常 |

**自定义 Promise Type 示例**:

```cpp
template <typename T>
class task {
public:
    struct promise_type {
        T value;
        std::exception_ptr exception;

        task get_return_object() {
            return task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept { return {}; }

        std::suspend_always final_suspend() noexcept { return {}; }

        void return_value(T v) { value = std::move(v); }

        void unhandled_exception() { exception = std::current_exception(); }
    };

    // ...
};
```

### Coroutine Handle

`std::coroutine_handle` 是协程的类型擦除句柄，用于手动控制协程的恢复和销毁。

**核心操作**:

```cpp
// 从 promise 获取句柄
auto handle = std::coroutine_handle<promise_type>::from_promise(promise);

// 从地址创建句柄
auto handle = std::coroutine_handle<>::from_address(ptr);

// 恢复协程
handle.resume();

// 销毁协程
handle.destroy();

// 检查是否完成
bool done = handle.done();

// 访问 promise
promise_type& p = handle.promise();
```

**重要注意事项**:

- `destroy()` 只能在协程处于最终挂起状态时调用
- `resume()` 不能在已完成的协程上调用
- 协程句柄是裸指针，需要手动管理生命周期

### Awaitable 与 Awaiter

**Awaitable**: 可以被 `co_await` 等待的对象。Awaitable 通过以下方式获得 Awaiter：

1. 成员函数 `operator co_await()`
2. 非成员函数 `operator co_await(awaitable)`
3. 如果以上都不存在，Awaitable 本身就是 Awaiter

**Awaiter**: 实现了三个核心方法的对象：
- `await_ready()`: 检查是否需要挂起
- `await_suspend(handle)`: 执行挂起逻辑
- `await_resume()`: 返回操作结果

**标准 Awaiter**:

```cpp
// std::suspend_always: 总是挂起
struct suspend_always {
    bool await_ready() const noexcept { return false; }
    void await_suspend(std::coroutine_handle<>) const noexcept {}
    void await_resume() const noexcept {}
};

// std::suspend_never: 从不挂起
struct suspend_never {
    bool await_ready() const noexcept { return true; }
    void await_suspend(std::coroutine_handle<>) const noexcept {}
    void await_resume() const noexcept {}
};
```

### Boost.Asio Awaitable

Prism 使用 `boost::asio::awaitable<T>` 作为协程返回类型。这是 Boost.Asio 提供的协程框架，深度集成 I/O 事件循环。

**核心特性**:

```cpp
namespace net = boost::asio;

// 定义返回 awaitable 的协程
net::awaitable<int> async_read_some(tcp::socket& socket, net::mutable_buffer buffer) {
    auto bytes = co_await socket.async_read_some(buffer, net::use_awaitable);
    co_return static_cast<int>(bytes);
}

// 获取当前执行器
net::awaitable<void> example() {
    auto executor = co_await net::this_coro::executor;
    auto cancellation_signal = co_await net::this_coro::cancellation_state;

    // 使用执行器创建定时器
    net::steady_timer timer(executor);
    timer.expires_after(std::chrono::seconds(1));
    co_await timer.async_wait(net::use_awaitable);
}

// 启动协程
net::co_spawn(io_context, example(), net::detached);
// 或带完成回调
net::co_spawn(io_context, example(), [](std::exception_ptr e) {
    if (e) std::rethrow_exception(e);
});
```

**`net::use_awaitable` 的工作原理**:

`net::use_awaitable` 是一个完成令牌（completion token），告诉 Boost.Asio 以协程方式完成异步操作。它会创建一个 Awaiter，当异步操作完成时恢复协程。

```cpp
// 内部实现伪代码
template <typename Op>
auto async_operation(Op&& op, net::use_awaitable_t) {
    struct awaiter {
        Op& op;
        std::optional<result_type> result;

        bool await_ready() { return false; }

        void await_suspend(std::coroutine_handle<> handle) {
            op([this, handle](auto... args) {
                result = {args...};
                handle.resume();
            });
        }

        auto await_resume() { return *result; }
    };
    return awaiter{op};
}
```

### 异常处理

协程中的异常会被捕获并存储在 `promise_type` 中：

```cpp
net::awaitable<void> may_throw() {
    throw std::runtime_error("协程异常");
    co_return;
}

net::awaitable<void> caller() {
    try {
        co_await may_throw();
    } catch (const std::runtime_error& e) {
        std::cout << "捕获异常: " << e.what() << "\n";
    }
}
```

当协程抛出异常时：
1. 异常被 `promise_type::unhandled_exception()` 捕获
2. 异常通过 `std::current_exception()` 存储
3. 当协程被 `co_await` 时，存储的异常会被重新抛出

### 取消支持

C++20 协程本身没有内置取消机制，但 Boost.Asio 提供了取消支持：

```cpp
net::awaitable<void> cancellable_operation() {
    auto cancel_state = co_await net::this_coro::cancellation_state;

    net::steady_timer timer(co_await net::this_coro::executor);
    timer.expires_after(std::chrono::seconds(10));

    // 检查取消状态
    if (cancel_state.cancelled() != net::cancellation_type::none) {
        co_return;  // 已取消
    }

    co_await timer.async_wait(net::use_awaitable);
}

// 取消协程
net::cancellation_signal signal;
net::co_spawn(io_context, cancellable_operation(), signal.slot());
signal.emit(net::cancellation_type::all);  // 发送取消信号
```

## 在 Prism 中的应用

Prism 是一个高性能代理服务器，采用**纯协程架构**，所有异步操作均通过 `co_await` 实现，避免了阻塞和回调地狱。

### 协程架构设计

Prism 的架构基于以下原则：

1. **无阻塞**: 所有 I/O 操作使用异步 API
2. **线性流程**: 协程让异步代码读起来像同步代码
3. **错误处理**: 通过 `fault::code` 返回错误码，避免异常开销
4. **生命周期**: 通过 `shared_ptr` 按值捕获保证对象存活

### 典型协程使用场景

#### 会话处理

```cpp
// agent/session/session.hpp
net::awaitable<void> session::run() {
    // 1. 协议识别
    auto result = co_await recognition::recognize(transport_, buffer_);

    if (result.failed()) {
        co_return;
    }

    // 2. 获取处理器
    auto handler = dispatch::registry::global().create(result.detected);

    // 3. 处理请求
    auto outcome = co_await handler->handle(transport_, result, account_);

    // 4. 清理
    co_return;
}
```

#### TLS 握手

```cpp
// stealth/reality/handshake.cpp
net::awaitable<std::pair<fault::code, shared_transmission>>
reality_handshake::perform(shared_transmission transport, const config& cfg) {
    // 1. 读取 ClientHello
    auto [ec, bytes] = co_await transport->read_some(buffer);
    if (ec) {
        co_return {ec, nullptr};
    }

    // 2. 解析 ClientHello
    auto features = parse_clienthello(buffer);

    // 3. 执行 Reality 握手
    auto result = co_await do_handshake(transport, features, cfg);

    co_return result;
}
```

#### 连接池

```cpp
// channel/connection/pool.cpp
net::awaitable<std::pair<fault::code, shared_connection>>
connection_pool::acquire(const endpoint& ep) {
    // 1. 检查空闲连接
    if (auto conn = try_acquire_idle(ep)) {
        co_return {fault::code::success, conn};
    }

    // 2. 创建新连接
    auto [ec, conn] = co_await create_connection(ep);
    if (ec) {
        co_return {ec, nullptr};
    }

    co_return {fault::code::success, conn};
}
```

#### DNS 解析

```cpp
// resolve/dns/detail/resolver.cpp
net::awaitable<std::pair<fault::code, std::vector<ip_address>>>
resolver::resolve(const std::string& domain) {
    // 1. 检查缓存
    if (auto cached = cache_.lookup(domain)) {
        co_return {fault::code::success, *cached};
    }

    // 2. 查询上游
    auto [ec, addrs] = co_await query_upstream(domain);
    if (ec) {
        co_return {ec, {}};
    }

    // 3. 更新缓存
    cache_.store(domain, addrs);

    co_return {fault::code::success, addrs};
}
```

### 协程纯度保证

Prism 在协程中禁止使用任何阻塞操作：

| 禁止 | 替代方案 |
|------|----------|
| `std::mutex` / `std::lock_guard` | `std::atomic`、`strand`、`concurrent_channel` |
| `std::this_thread::sleep_for()` | `net::steady_timer::async_wait()` |
| 阻塞 socket read/write | `async_read_some`/`async_write_some` |
| `::getaddrinfo()` 同步 DNS | `resolver.async_resolve()` |
| `std::future::get()` / `wait()` | `co_await` 异步结果 |
| `while (!flag) {}` 忙等待 | `co_await` 异步等待 + 通知机制 |

### 生命周期安全

在协程中使用 `co_spawn` 启动独立协程时，必须保证对象生命周期：

```cpp
// 错误示例：悬空引用
net::awaitable<void> bad_example() {
    std::string local = "hello";
    net::co_spawn(executor, [&local]() -> net::awaitable<void> {
        // local 已被销毁，悬空引用！
        std::cout << local << "\n";
        co_return;
    }, net::detached);
}

// 正确示例：按值捕获
net::awaitable<void> good_example() {
    auto self = shared_from_this();  // 捕获 shared_ptr
    net::co_spawn(executor, [self]() -> net::awaitable<void> {
        co_await self->async_operation();
        co_return;
    }, net::detached);
}
```

## 最佳实践

### 1. 始终使用 co_await 调用异步操作

```cpp
// 推荐：协程风格
auto bytes = co_await socket.async_read_some(buffer, net::use_awaitable);

// 避免：回调风格
socket.async_read_some(buffer, [](auto ec, auto bytes) {
    // 回调地狱
});
```

### 2. 使用 this_coro 获取执行器和取消状态

```cpp
net::awaitable<void> operation() {
    auto executor = co_await net::this_coro::executor;
    auto cancel_state = co_await net::this_coro::cancellation_state;

    // 使用执行器创建异步对象
    net::steady_timer timer(executor);

    // 检查取消状态
    if (cancel_state.cancelled() != net::cancellation_type::none) {
        co_return;
    }
}
```

### 3. 协程中的错误处理

```cpp
// 方式一：返回错误码（Prism 风格）
net::awaitable<std::pair<fault::code, result>> operation() {
    auto [ec, data] = co_await async_read();
    if (ec) {
        co_return {ec, {}};
    }
    co_return {fault::code::success, process(data)};
}

// 方式二：异常处理
net::awaitable<void> operation_with_exception() {
    try {
        auto data = co_await async_read();
        // 处理数据
    } catch (const std::exception& e) {
        // 处理异常
    }
}
```

### 4. 避免协程中的阻塞

```cpp
// 错误：阻塞协程
net::awaitable<void> bad() {
    std::this_thread::sleep_for(std::chrono::seconds(1));  // 阻塞！
    co_return;
}

// 正确：异步等待
net::awaitable<void> good() {
    net::steady_timer timer(co_await net::this_coro::executor);
    timer.expires_after(std::chrono::seconds(1));
    co_await timer.async_wait(net::use_awaitable);
}
```

### 5. 正确管理协程生命周期

```cpp
class session : public std::enable_shared_from_this<session> {
public:
    net::awaitable<void> run() {
        auto self = shared_from_this();  // 延长生命周期
        co_await do_something();
    }

    void start() {
        net::co_spawn(executor_, run(), net::detached);
    }
};
```

### 6. 使用 strand 保证线程安全

```cpp
net::awaitable<void> thread_safe_operation() {
    auto strand = net::make_strand(co_await net::this_coro::executor);

    // 多个协程访问共享资源时使用 strand
    co_await net::post(strand, net::use_awaitable);
    // 现在在 strand 中执行，线程安全
}
```

### 7. 取消协程的正确方式

```cpp
net::awaitable<void> cancellable() {
    auto cancel_slot = co_await net::this_coro::cancellation_slot;

    // 注册取消处理器
    cancel_slot.assign([](net::cancellation_type type) {
        // 执行清理
    });

    // 或者在异步操作中支持取消
    net::steady_timer timer(co_await net::this_coro::executor);
    timer.expires_after(std::chrono::seconds(10));
    co_await timer.async_wait(net::bind_cancellation_slot(cancel_slot, net::use_awaitable));
}
```

## 常见问题

### Q1: 协程和线程有什么区别？

协程是用户态的轻量级执行单元，由程序员控制调度；线程是内核态的执行单元，由操作系统调度。协程的挂起和恢复不涉及内核切换，开销远小于线程。

| 特性 | 协程 | 线程 |
|------|------|------|
| 调度方式 | 用户态协作式 | 内核态抢占式 |
| 切换开销 | 纳秒级 | 微秒级 |
| 栈大小 | 动态分配（通常几 KB） | 固定（通常 1MB+） |
| 数量限制 | 百万级 | 数百到数千 |
| 阻塞影响 | 仅当前协程 | 整个线程 |

### Q2: 什么是协程帧泄漏？

协程帧在堆上分配，如果协程在挂起状态下被遗忘，协程帧不会被释放，造成内存泄漏。常见原因：

1. **未恢复协程**: 创建协程后从未调用 `resume()`
2. **循环引用**: 协程帧持有 `shared_ptr`，`shared_ptr` 又持有协程帧
3. **取消不当**: 取消协程时未正确清理

解决方案：
- 使用 RAII 管理 `coroutine_handle`
- 确保协程有明确的退出路径
- 使用 `cancellation_signal` 正确取消

### Q3: co_await 的返回值类型是什么？

`co_await expr` 的返回值类型由 `await_resume()` 的返回值类型决定：

```cpp
struct int_awaiter {
    int value;
    bool await_ready() { return true; }
    void await_suspend(std::coroutine_handle<>) {}
    int await_resume() { return value; }  // 返回 int
};

net::awaitable<void> example() {
    int result = co_await int_awaiter{42};  // result 是 int
}
```

对于 `net::use_awaitable`，返回值类型取决于异步操作的完成签名：
- `async_read_some` 返回 `size_t`（读取的字节数）
- `async_write_some` 返回 `size_t`（写入的字节数）
- `async_connect` 返回 `void`

### Q4: 如何调试协程？

调试协程的挑战在于协程帧在堆上，调用栈可能不连续。建议：

1. **日志追踪**: 在协程的关键点添加日志
2. **协程 ID**: 使用 `std::coroutine_handle::address()` 作为协程标识
3. **GDB 调试**: 使用 `info coroutine` 命令（GCC 支持）
4. **Boost.Asio 调试**: 定义 `BOOST_ASIO_ENABLE_HANDLER_TRACKING` 宏

```cpp
// 定义跟踪宏
#define BOOST_ASIO_ENABLE_HANDLER_TRACKING

// 启用后会在 stderr 输出协程追踪信息
// @asio|1234567890.123456|0*1|co_spawn
// @asio|1234567890.234567|0>1|async_read
```

### Q5: 协程可以递归调用吗？

可以，但需要注意栈溢出问题。无栈协程的递归深度理论上没有限制，但每次递归都会创建新的协程帧：

```cpp
net::awaitable<int> factorial(int n) {
    if (n <= 1) co_return 1;
    auto result = co_await factorial(n - 1);
    co_return n * result;
}

// 警告：深度递归可能耗尽内存
// 建议：改用循环或尾递归优化
```

### Q6: co_spawn 和直接调用协程有什么区别？

直接调用协程函数会立即返回一个 awaitable 对象，协程在第一个挂起点挂起；`co_spawn` 会在指定执行器上启动协程，并可以附加完成回调：

```cpp
// 直接调用：协程在当前执行器上运行
auto awaitable = my_coroutine();  // 协程已开始执行
co_await awaitable;  // 等待完成

// co_spawn：在指定执行器上运行
net::co_spawn(other_executor, my_coroutine(), net::detached);
// 立即返回，不等待完成

// 带完成回调
net::co_spawn(executor, my_coroutine(), [](std::exception_ptr e) {
    if (e) std::rethrow_exception(e);
    std::cout << "协程完成\n";
});
```

### Q7: 如何处理协程中的超时？

使用 `deadline_timer` 和 `cancellation_signal`：

```cpp
net::awaitable<std::string> fetch_with_timeout(int seconds) {
    auto executor = co_await net::this_coro::executor;
    net::steady_timer timer(executor);
    timer.expires_after(std::chrono::seconds(seconds));

    net::cancellation_signal cancel;

    // 启动异步操作
    auto result = co_await (
        async_fetch() || timer.async_wait(net::use_awaitable)
    );

    if (result.index() == 1) {
        // 超时
        cancel.emit(net::cancellation_type::all);
        co_return "timeout";
    }

    co_return std::get<0>(result);
}
```

### Q8: 协程性能如何？

协程的开销主要来自：
1. 协程帧分配（堆分配）
2. 状态机转换（跳转）
3. 上下文保存（寄存器）

但这些开销远小于线程切换。实测数据：
- 协程挂起/恢复：~50ns
- 线程切换：~1-10us
- 协程内存占用：~200-500 字节
- 线程内存占用：~1MB（栈空间）

在 I/O 密集型场景，协程可以轻松支持百万级并发连接。

## 参考资料

- [C++20 Coroutines (cppreference)](https://en.cppreference.com/w/cpp/language/coroutines)
- [Boost.Asio Coroutines](https://www.boost.org/doc/libs/release/doc/html/boost_asio/overview/composition/coroutines.html)
- [C++ Coroutines: Under the covers](https://devblogs.microsoft.com/oldnewthing/20210407-00/?p=105104)
- [Lewis Baker's Coroutine Articles](https://lewissbaker.github.io/)

## 相关知识

- [[ref/programming/boost-asio|Boost.Asio]] — 异步 I/O 框架
- [[ref/programming/constexpr|constexpr 编译期计算]] — 编译期优化
- [[core/agent/session/session|会话管理]] — 协程在会话中的应用
- [[core/stealth/executor|方案执行器]] — 协程握手实现