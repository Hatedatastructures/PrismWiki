---
title: Boost.Asio Coroutine
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [programming, boost, asio, coroutine]
---

# Boost.Asio 协程

> Boost.Asio 的异步模型与 C++20/23 协程整合原理。
> 最后更新：2026-05-17

---

## Asio 异步模型

### Proactor 模式

Asio 采用 Proactor 模式：

```
 Initiator           Proactor          Asynchronous Operation
    │                   │                      │
    │ start async op    │                      │
    │──────────────────>│─────────────────────>│
    │                   │                      │
    │                   │   operation completes│
    │                   │<─────────────────────│
    │                   │                      │
    │ completion handler│                      │
    │<──────────────────│                      │
```

### 异步操作类型

| 操作 | 方法 | 说明 |
|------|------|------|
| 读取 | `async_read_some` | 读取部分数据 |
| 写入 | `async_write_some` | 写入部分数据 |
| 连接 | `async_connect` | 异步连接 |
| 接受 | `async_accept` | 异步接受连接 |
| 定时 | `async_wait` | 定时器等待 |

---

## 协程适配

### awaitable<T>

Asio 提供 `awaitable<T>` 作为协程返回类型：

```cpp
// 协程函数签名
asio::awaitable<void> handle_session(tcp::socket socket);

asio::awaitable<size_t> read_data(tcp::socket& socket);
asio::awaitable<void> write_data(tcp::socket& socket, std::span<uint8_t> data);
```

### co_spawn

启动协程：

```cpp
// 启动协程
asio::co_spawn(
    io_context,
    handle_session(std::move(socket)),
    asio::detached  // 或 use_awaitable
);
```

### co_await

等待异步操作：

```cpp
asio::awaitable<void> handle_session(tcp::socket socket) {
    // 等待读取
    size_t n = co_await socket.async_read_some(
        asio::buffer(data),
        asio::use_awaitable
    );
    
    // 等待写入
    co_await asio::async_write(
        socket,
        asio::buffer(response),
        asio::use_awaitable
    );
}
```

---

## 执行上下文

### io_context

事件循环核心：

```cpp
asio::io_context ctx;
// 添加工作
asio::co_spawn(ctx, task1(), asio::detached);
asio::co_spawn(ctx, task2(), asio::detached);
// 运行
ctx.run();
```

### strand

序列化执行：

```cpp
asio::strand<asio::io_context::executor_type> strand(ctx.get_executor());

// 保证顺序
asio::co_spawn(strand, task1(), asio::detached);
asio::co_spawn(strand, task2(), asio::detached);
```

---

## 定时器

### steady_timer

高精度定时器：

```cpp
asio::awaitable<void> with_timeout(tcp::socket& socket) {
    asio::steady_timer timer(co_await asio::this_coro::executor);
    timer.expires_after(std::chrono::seconds(5));
    
    // 等待定时器或操作
    auto result = co_await timer.async_wait(asio::use_awaitable);
}
```

### 超时模式

```cpp
asio::awaitable<void> timeout_read(tcp::socket& socket) {
    asio::steady_timer timer(co_await asio::this_coro::executor);
    timer.expires_after(std::chrono::milliseconds(300));
    
    // 并发等待
    auto order = co_await asio::experimental::make_parallel_group({
        socket.async_read_some(asio::buffer(data), asio::use_awaitable),
        timer.async_wait(asio::use_awaitable)
    }).async_wait(asio::experimental::wait_for_one(), asio::use_awaitable);
    
    if (order[1] == 0) { // timer completed first
        socket.close();
        throw timeout_error();
    }
}
```

---

## 错误处理

### try-catch

协程内异常处理：

```cpp
asio::awaitable<void> safe_handler(tcp::socket socket) {
    try {
        co_await handle_data(socket);
    } catch (const std::exception& e) {
        log_error(e.what());
        socket.close();
    }
}
```

### 错误码

使用 `asio::error_code`：

```cpp
asio::awaitable<void> error_code_read(tcp::socket& socket) {
    asio::error_code ec;
    size_t n = co_await socket.async_read_some(
        asio::buffer(data),
        asio::redirect_error(asio::use_awaitable, ec)
    );
    
    if (ec) {
        // 处理错误
        return;
    }
}
```

---

## Prism 应用

Prism 使用 Asio 协程实现：

| 模块 | 使用方式 | 详见 |
|------|----------|------|
| Agent | 监听接受协程 | [[core/agent/front/listener|Listener]] |
| Session | 会话处理协程 | [[core/session/session|Session]] |
| Channel | 连接建立协程 | [[core/channel/overview|Channel]] |
| Pipeline | 协议处理协程 | [[core/pipeline/overview|Pipeline]] |

---

## 相关参考

- [[cpp23-coroutine|C++23 协程]] — 协程原理
- [[dev/coding/coroutine|协程约定]] — Prism 协程规范
- [[core/agent/front/listener|Listener 实现]] — Asio 使用实例

---

## 进一步阅读

- Boost.Asio 文档: https://www.boost.org/doc/libs/release/doc/html/boost_asio.html
- C++20 协程提案: https://en.cppreference.com/w/cpp/language/coroutines