---
title: C++23 协程
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [cpp, coroutine, async, networking]
sources:
  - I:/code/Prism/README.md
  - I:/code/Prism/CLAUDE.md
confidence: high
---

# C++23 协程

C++23 协程是一种异步编程模型，使用 `co_await`、`co_yield`、`co_return` 关键字实现协程函数。

## 概述

C++23 协程是 C++20 引入的协程特性的进一步完善，提供了更简洁的异步编程模型。在 Prism 中，协程用于替代传统的回调嵌套，实现全链路异步。

## 核心概念

### 协程关键字

- `co_await` — 挂起协程，等待异步操作完成
- `co_yield` — 生成值并挂起协程（用于生成器）
- `co_return` — 从协程返回值

### 协程类型

- **Task** — 表示一个异步任务，可以 `co_await` 等待完成
- **Generator** — 生成器，可以 `co_yield` 产出值
- **Lazy** — 惰性求值，只在需要时执行

### 协程状态

- **挂起** — 协程暂停执行，保存状态
- **恢复** — 协程继续执行，恢复状态
- **完成** — 协程执行完毕，返回结果

## 在 Prism 中的应用

Prism 使用 C++23 协程实现：

1. **全链路异步** — 每个连接由独立协程驱动，无回调嵌套
2. **零拷贝** — 数据直接在协程间传递，无需复制
3. **非阻塞 I/O** — 协程挂起等待 I/O，不阻塞线程
4. **并发处理** — 多个协程可以并发执行，提高吞吐量

## 示例代码

```cpp
// 简单的协程示例
Task<std::string> fetchData(std::string url) {
    auto response = co_await httpClient.get(url);
    co_return response.body();
}

// 在 Prism 中的应用
Task<void> handleConnection(Socket socket) {
    auto data = co_await socket.read();
    auto result = co_await process(data);
    co_await socket.write(result);
}
```

## 协程在代理引擎中的实际应用场景

### 每个连接一个协程

代理引擎为每个客户端连接创建一个独立协程，该协程负责整个连接的生命周期：从接受连接、协议握手、数据转发到连接关闭。协程内部的代码逻辑与同步代码几乎一致，但底层是异步执行的。

```
客户端连接 → spawn 协程 → 协程内执行：
  co_await 接受连接
  co_await 协议握手
  循环 { co_await 读数据; co_await 转发; }
  连接关闭，协程销毁
```

这种模型的优势在于每个协程的状态机由编译器自动生成，开发者写的是线性代码，运行时是异步调度。

### 异步 DNS 解析

DNS 解析是代理引擎的高频操作。使用协程可以将 DNS 查询包装为异步操作：

```cpp
Task<IPAddress> resolveDNS(std::string hostname) {
    auto result = co_await dnsResolver.asyncResolve(hostname);
    co_return result;
}
```

协程在等待 DNS 响应时挂起，不阻塞线程，同一时间可以处理成千上万个并发 DNS 查询。

### 连接池管理

连接池中的空闲连接复用、健康检查、超时回收都可以用协程实现：

```cpp
Task<Connection> getConnection(Pool& pool) {
    auto conn = pool.tryAcquire();
    if (!conn) {
        co_await pool.waitForAvailable();  // 挂起等待可用连接
        conn = pool.acquire();
    }
    co_return conn;
}
```

## co_await 在 Boost.Asio 中的用法

Boost.Asio 是 Prism 使用的 I/O 框架，它原生支持 C++20/23 协程。常见的 `co_await` 用法：

- `co_await socket.async_read(buffer, use_awaitable)` — 异步读取
- `co_await socket.async_write(buffer, use_awaitable)` — 异步写入
- `co_await timer.async_wait(use_awaitable)` — 异步等待定时器
- `co_await acceptor.async_accept(use_awaitable)` — 异步接受连接
- `co_await co_spawn(strand, coroutine, use_awaitable)` — 在指定 strand 上启动协程

Boost.Asio 的 `awaitable<T>` 类型实现了 C++20 Awaitable 接口，可以直接与 `co_await` 配合使用。每个协程通过 `strand` 保证同一线程安全，避免数据竞争。

## 协程 vs 回调 vs 线程池的性能对比

| 特性       | 协程             | 回调        | 线程池           |
| -------- | -------------- | --------- | ------------- |
| 代码可读性    | 线性代码，接近同步风格    | 回调嵌套，难以维护 | 同步风格，但需要锁     |
| 内存开销     | 每个协程 ~256B-2KB | 几乎无额外开销   | 每个线程 ~8MB 栈   |
| 上下文切换    | 用户态切换，~10ns    | 无切换（事件驱动） | 内核态切换，~1-10μs |
| 并发能力     | 百万级并发          | 百万级并发     | 受限于线程数（通常数千）  |
| 调试难度     | 较低             | 高（回调地狱）   | 中等（死锁、竞态）     |
| CPU 缓存友好 | 好（连续内存布局）      | 好         | 差（线程切换刷新缓存）   |

在代理引擎场景下，协程在可读性和并发能力上优于回调和线程池，同时内存开销远低于线程池模型。

## 协程的内存开销

每个协程的内存开销取决于协程帧（coroutine frame）的大小，主要包含：

- **协程状态** — suspend point 的数量和类型
- **局部变量** — 协程函数内的所有局部变量
- **Promise 对象** — 协程的 promise_type 实例
- **对齐填充** — 内存对齐的额外字节

典型代理连接协程的帧大小约 256B-2KB。在 Prism 中，由于使用了 PMR 内存池（[[PMR-Memory-Pool]]），协程帧直接从线程独占池分配，避免了堆分配开销。

对于同时处理 10 万个并发连接的场景，协程总内存约 25MB-200MB，而等价的线程池方案需要 800GB+ 栈空间（每线程 8MB）。

## Prism 中协程的具体应用示例

在 [[agent]] 中，协程贯穿整个数据路径：

1. **接受连接** — `co_await acceptor.async_accept()` 接受新连接，spawn 新协程
2. **协议识别** — `co_await readFirstPacket()` 读取首包，判断协议类型
3. **TLS 握手** — `co_await tlsLayer.handshake()` 执行 TLS 握手（Reality 等）
4. **协议解析** — `co_await parseProtocolHeader()` 解析代理协议头
5. **DNS 解析** — `co_await dnsResolve(targetHost)` 异步解析目标地址
6. **建立出站连接** — `co_await connectToTarget()` 建立到目标服务器的连接
7. **双向转发** — 两个协程分别负责上行和下行数据转发
8. **连接关闭** — 资源归还内存池，协程销毁

每一步都是 `co_await`，整条链路是线性代码，没有回调嵌套。

## 优势

- **简洁** — 代码结构清晰，无回调嵌套
- **高效** — 协程切换开销小，性能接近同步代码
- **可维护** — 异步代码更容易理解和调试
- **可组合** — 协程可以轻松组合，实现复杂逻辑

## 相关页面

- [[agent]] — 使用 C++23 协程的服务端代理引擎
- [[PMR-Memory-Pool]] — 协程中的内存管理
- [[TCP]] — 传输层协议，协程 I/O 的基础
- [[Proxy-Protocols]] — 代理协议，协程处理的对象
- [[Mihomo-Clash-Config]] — mihomo 客户端配置
