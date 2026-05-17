---
title: 协程约定
description: Prism 协程编程规范，包括 awaitable 使用、协程纯度要求和禁止事项
---

# 协程约定

Prism 采用 **C++23 纯协程架构**，所有异步操作均基于 Boost.Asio 协程实现。本文档定义协程编程的约定和禁止事项。

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

## 相关文档

- [[overview]] - 编码规范概览
- [[lifecycle]] - 生命周期安全指南
- [[/dev/cpp23-coroutines]] - C++23 协程详解
- [[/dev/modules]] - 模块结构