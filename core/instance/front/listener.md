---
title: listener — 前端监听器
layer: core
source: include/prism/instance/front/listener.hpp
created: 2026-05-17
tags: [agent, front, listener, accept, affinity]
---

# listener — 前端监听器

> 源码: `include/prism/instance/front/listener.hpp`

## 概述

`listener` 是 Prism 代理服务器的入站阶段入口组件，位于 **Front Layer** 的最顶层。负责监听指定端口、接受入站 TCP 连接，并将连接分发给负载均衡器进行后续处理。

### 核心职责

| 职责 | 说明 |
|------|------|
| TCP 监听 | 绑定地址端口，持续异步接受连接 |
| 亲和性计算 | 根据客户端 IP 计算一致性哈希值 |
| 连接分发 | 调用 balancer 的 `select()` 和 `dispatch()` |
| 反压处理 | 系统过载时延迟接受新连接 |

### 调用关系

```
[main.cpp] --构造-> [[listener]] --listen()--> accept_loop协程
                                  |
                                  +-> make_affinity()
                                  |
                                  +-> [[balancer]].select()
                                          |
                                          +-> [[balancer]].dispatch()
                                                  |
                                                  +-> [[worker]].dispatch_socket()
```

---

## 类接口

| 方法 | 说明 |
|------|------|
| `listener(const psm::config&, balancer&)` | 构造监听器，初始化 io_context/acceptor/缓冲区/反压参数 |
| `listen()` | 启动异步接受循环，阻塞运行 io_context |
| `stop()` | 关闭 acceptor，停止 io_context，优雅关闭 |

### 私有方法

| 方法 | 说明 |
|------|------|
| `make_affinity(endpoint) -> uint64` | 根据客户端 IP 计算亲和性哈希（静态） |
| `accept_loop() -> awaitable<void>` | 异步接受循环协程 |

### 成员变量

| 成员 | 类型 | 说明 |
|------|------|------|
| `ioc_` | `net::io_context` | 独立事件循环，与 worker 隔离 |
| `acceptor_` | `tcp::acceptor` | TCP 接受器，绑定监听端口 |
| `dispatcher_` | `balancer &` | 负载均衡器引用 |
| `buffer_size_` | `uint32` | Socket 收发缓冲区大小（默认 256KB） |
| `backpressure_delay_` | `milliseconds` | 反压延迟时间（固定 2ms） |

---

## 构造流程

构造函数根据配置执行：

1. 初始化 `io_context(1)` — 单线程独立事件循环
2. 解析监听地址：`localhost` -> `127.0.0.1`，空或 `0.0.0.0` -> 所有接口，其他无效地址抛出 `std::system_error`
3. 配置 acceptor：`open()` -> `reuse_address(true)` -> 设置收发缓冲区大小 -> `bind()` -> `listen()`
4. 设置反压延迟 2ms

---

## make_affinity() 亲和性计算

根据客户端 IP 地址计算哈希值，用于一致性选择：

- **IPv4**: 直接返回 32 位地址值（如 `192.168.1.1` -> `0xC0A80101`）
- **IPv6**: 高低 8 字节 XOR 折叠为 64 位

设计意图：相同客户端 IP 总是产生相同的亲和性值，balancer 基于此值进行一致性哈希选择，同一客户端连接倾向于分发到同一 worker。端口号未参与计算，仅使用 IP 地址。

---

## accept_loop() 协程

持续接受入站连接的核心循环：

### 流程

```
获取协程执行器 + 创建 steady_timer
初始化指数退避参数: min=10ms, max=5120ms
    |
    v
[循环]
    |
    +-> async_accept()
    |       |
    |       +-- 致命错误(too_many_files_open, not_enough_memory)
    |       |       -> 指数退避: delay = min(delay*2, 5120ms)
    |       |       -> co_await timer, continue
    |       |
    |       +-- 普通错误
    |       |       -> 固定 10ms 延迟, continue
    |       |
    |       +-- 成功
    |               -> 重置退避延迟
    |               -> make_affinity(remote_endpoint)
    |               -> dispatcher_.select(affinity) -> decision
    |               -> if decision.backpressure: 等待 2ms
    |               -> dispatcher_.dispatch(worker_index, socket)
```

### 错误处理

| 错误类型 | 处理策略 | 延迟 |
|----------|----------|------|
| `too_many_files_open` | 指数退避 | 10ms -> 5120ms |
| `not_enough_memory` | 指数退避 | 10ms -> 5120ms |
| 其他错误 | 固定延迟 | 10ms |
| 地址解析失败 | 抛出异常 | `std::system_error` |
| 端口被占用 | 抛出异常 | `std::system_error` |

---

## 设计决策

### 为什么使用独立 io_context？

**问题**: 如果监听和 worker 共享 io_context，worker 过载会阻塞 accept 循环。

**选择**: listener 使用独立 `io_context(1)`，与 worker 线程完全隔离。

**后果**: socket 跨线程转移增加少量开销，但保证监听不受 worker 负载影响，accept 能力始终稳定。

### 为什么使用协程而非回调？

**问题**: 传统回调模式的 accept 循环在错误处理和状态管理上容易产生回调地狱。

**选择**: 使用 Boost.Asio 协程（`co_await async_accept`）实现接受循环。

**后果**: 代码流程清晰，错误处理统一，与 Prism 纯协程架构一致。

### 为什么反压使用固定 2ms 延迟？

**问题**: 系统过载时，拒绝连接会导致客户端重连风暴，加剧过载。

**选择**: 当 balancer 返回反压信号时，延迟 2ms 后继续接受（不拒绝）。

**后果**: 2ms 足够短不影响正常流量，也足够长让过载 worker 有恢复机会，平滑处理而非暴力拒绝。

### 为什么 make_affinity 只用 IP 不用端口？

**问题**: 亲和性需要保证同一客户端的多个连接分发到同一 worker。

**选择**: 只使用客户端 IP 地址计算哈希，不包含端口号。

**后果**: 同一客户端的所有连接（不同源端口）具有相同亲和性值，确保会话保持。端口号变化频繁（每次连接不同），纳入计算反而破坏一致性。

---

## 调用链

### 上游调用者

| 调用者 | 调用时机 |
|--------|----------|
| `main.cpp` | 启动时构造实例 |
| `main.cpp` | 监听线程中调用 `listen()` |

### 完整调用链

```
main.cpp  listener service_listener(config_ref, dispatcher)
    |
    v
main.cpp  service_listener.listen()
    |
    v
listener.cpp  net::co_spawn(ioc_, accept_loop(), net::detached)
    |
    v
listener.cpp  dispatcher_.select(affinity)
    |               -> balancer::select()
    |               -> balancer::score()
    |               -> balancer::refresh_state()
    |
    v
listener.cpp  dispatcher_.dispatch(worker_index, socket)
                    -> balancer::dispatch()
                    -> worker_binding.dispatch(socket)
                    -> [[worker]].dispatch_socket()
```

---

## 相关文档

- [[core/instance/front/balancer|balancer]] -- 负载均衡器
- [[core/instance/worker/worker|worker]] -- 工作线程
- [[core/instance/config|config]] -- 配置详解
- [[core/startup]] -- 启动流程
