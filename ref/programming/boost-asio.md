---
title: Boost.Asio 与 Prism 协程
created: 2026-05-17
updated: 2026-05-29
layer: ref
tags: [programming, boost, asio, coroutine]
---

# Boost.Asio 与 Prism 协程

Prism 基于 Boost.Asio 构建纯协程架构，所有 I/O 操作均为异步。本文聚焦 Prism 使用的 Asio 模式和约定，而非通用 Asio 教程。

## Prism 使用的 Asio 模式

### 核心模式表

| 模式 | Prism 用法 | 示例位置 |
|------|-----------|---------|
| `use_awaitable` | 所有 async 操作的默认 completion token | transport、stealth、protocol |
| `redirect_error` | 将错误码从异常转为输出参数 | tunnel、seal、handshake |
| `co_spawn` | 启动独立协程（detached） | listener 分发、launch 会话 |
| `steady_timer` | 超时保护、竞速取消 | Reality 握手超时、Happy Eyeballs |
| `this_coro::executor` | 获取当前协程的执行器 | 所有需要 timer 的协程 |
| `detached` | 无人监听的协程完成 | 后台任务、日志刷入 |

### redirect_error 模式

Prism 热路径中禁止异常，使用 `redirect_error` 将 Asio 错误从异常转为错误码：

```cpp
std::error_code ec;
auto n = co_await socket.async_read_some(
    net::buffer(buf),
    net::redirect_error(net::use_awaitable, ec));
if (ec) {
    // 处理错误，不抛异常
}
```

### deadline 竞速模式

Reality 握手和 Happy Eyeballs 使用定时器与操作竞速：

```cpp
net::steady_timer deadline(inbound->executor(), std::chrono::seconds(30));
auto on_deadline = [&inbound](const boost::system::error_code &ec) {
    if (!ec) inbound->cancel();
};
deadline.async_wait(std::move(on_deadline));
// ... 执行握手 ...
deadline.cancel();  // 成功时取消定时器
```

## Prism 协程约定

### 每线程 io_context

每个 worker 线程拥有独立的 `io_context`，避免跨线程调度：

```
worker_0: io_context → session_0a, session_0b, ...
worker_1: io_context → session_1a, session_1b, ...
```

好处：同一线程上的协程无竞争，`strand` 仅在跨线程通信时需要。

### 禁止 mutex

协程中禁止 `std::mutex` / `std::lock_guard`。替代方案：

| 场景 | 替代方案 |
|------|----------|
| 计数器/标志位 | `std::atomic` + `memory_order_relaxed` |
| 跨线程消息 | `net::post()` 到目标 executor |
| 生产者-消费者 | `concurrent_channel` |
| 互斥访问 | `strand` 序列化 |

详见 [[dev/coding/coroutine|协程约定]]。

### co_spawn 保活

`co_spawn` 的 lambda 必须按值捕获 `self`（shared_ptr），保持对象存活：

```cpp
net::co_spawn(executor,
    [self = shared_from_this()]() -> net::awaitable<void> {
        co_await self->process();
    },
    net::detached);
```

### co_await 后引用失效

`co_await` 挂起恢复后，裸指针、迭代器、引用可能失效。必须重新获取：

```cpp
// 恢复后容器可能已被修改
co_await async_op();
auto it = vec.begin() + offset;  // 重新获取，不能用旧的 it
```

详见 [[dev/coding/lifecycle|生命周期管理]]。

### net::post 跨线程

从线程 A 向线程 B 的 `io_context` 提交任务：

```cpp
net::post(target_executor, [self]() {
    // 在目标线程上执行
});
```

用于 balancer 将连接分发给目标 worker。

## Prism 特有的协程管道

### 六阶段流水线

```
listener → balancer → worker → session → recognition → protocol handler
```

每个阶段都是独立协程，通过 `co_spawn` 和 `co_await` 串联。

### 传输层抽象

所有 I/O 通过 `transport::transmission` 接口：

```cpp
struct transmission {
    virtual auto async_read_some(span<byte>, error_code&) -> awaitable<size_t> = 0;
    virtual auto async_write_some(span<const byte>, error_code&) -> awaitable<size_t> = 0;
};
```

实现类：`reliable`（TCP）、`encrypted`（TLS）、`seal`（Reality AEAD）、`mux_stream`（多路复用）。

## 常见模式速查

### 写入完整缓冲区

```cpp
co_await net::async_write(socket, net::buffer(data, size),
    net::redirect_error(net::use_awaitable, ec));
```

### 读取固定长度

```cpp
co_await net::async_read(socket, net::buffer(buf, expected_size),
    net::redirect_error(net::use_awaitable, ec));
```

### 带超时的操作

```cpp
net::steady_timer timer(co_await net::this_coro::executor);
timer.expires_after(std::chrono::seconds(5));
timer.async_wait([&](const boost::system::error_code&) { socket.cancel(); });
// ... async operation ...
timer.cancel();
```

## 相关参考

- [[dev/coding/coroutine|协程约定]] — Prism 协程规范（纯度要求、禁止事项）
- [[dev/coding/lifecycle|生命周期管理]] — co_await 后引用失效、co_spawn 保活
- [[core/architecture|架构]] — 六阶段流水线详解
- [[core/transport/transmission|Transport]] — 传输层抽象接口
- [Boost.Asio 文档](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html)
