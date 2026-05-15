---
title: "listener.hpp — 前端代理连接监听器"
source: "include/prism/agent/front/listener.hpp"
module: "agent"
type: api
tags: [agent, front, listener, 监听]
created: 2026-05-15
updated: 2026-05-15
related:
  - agent/front/balancer
  - agent/worker/worker
  - ref/network/tcp
---

# listener.hpp

> 源码: `include/prism/agent/front/listener.hpp` / `src/prism/agent/front/listener.cpp`
> 模块: [[agent|Agent]] / front

## 概述

实现前端代理的连接监听组件，负责绑定监听地址、接受入站连接并分发给负载均衡器。监听器采用 [[ref/programming/boost-asio|Boost.Asio]] 协程模式实现异步接受循环，支持反压机制：当 [[agent/front/balancer|负载均衡器]] 报告全局过载时，监听器将延迟接受新连接，避免系统过载恶化。每个新连接根据客户端端点计算亲和性值，确保同一客户端的连接倾向于分发至同一 [[agent/worker/worker|工作线程]]。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/config|config]] | 代理配置，提供监听地址、缓冲区大小等参数 |
| 依赖 | [[agent/front/balancer|balancer]] | 负载均衡器，用于选择目标工作线程并分发连接 |
| 依赖 | [[ref/programming/boost-asio|Boost.Asio]] | 异步 IO 框架，提供 acceptor、协程、定时器 |
| 被依赖 | main.cpp | 主线程构造 listener 并调用 listen() 启动监听 |

## 命名空间

`psm::agent::front`

---

## 类: listener

### 概述

前端代理连接监听器，负责监听指定端口并接受入站 TCP 连接，将接受的连接通过负载均衡器分发给后端工作线程。运行在独立的 `io_context` 中，与工作线程隔离以避免相互影响。

### 设计意图

监听器运行在独立的 `io_context` 中（构造时通过 `ioc_(1)` 指定单线程事件循环），与工作线程隔离以避免相互影响。接受循环采用协程实现，当检测到全局反压信号时，会主动延迟接受操作，为系统恢复预留缓冲时间。致命错误（文件描述符耗尽/内存不足）时采用指数退避策略，普通错误则短延迟后重试。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `net::io_context` | `ioc_` | 独立的 IO 上下文，构造时初始化为单线程模式 |
| `tcp::acceptor` | `acceptor_` | TCP 接受器，在构造函数中打开并绑定 |
| `balancer &` | `dispatcher_` | 负载均衡器引用，用于选择工作线程并分发连接 |
| `std::uint32_t` | `buffer_size_` | 套接字收发缓冲区大小，从配置中读取 |
| `std::chrono::milliseconds` | `backpressure_delay_` | 反压延迟时间，默认 2ms |

### 生命周期

监听器在主线程中构造，通过独立线程调用 `listen()` 阻塞运行事件循环。生命周期必须长于其引用的负载均衡器。构造完成后 acceptor 已绑定端口但尚未开始接受连接，需显式调用 `listen()` 启动。

---

## 函数文档

### listener() 构造函数

**功能说明**
根据配置初始化独立的 IO 上下文与 TCP 接受器，解析监听地址、设置套接字选项并绑定端口。

**签名**
```cpp
explicit listener(const psm::config &cfg, balancer &dispatcher);
```

**参数**

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `cfg` | `const psm::config &` | 代理配置，包含监听地址 (`agent.addressable.host/port`)、缓冲区大小 (`buffer.size`) 等参数 |
| `dispatcher` | `balancer &` | 负载均衡器引用，用于后续连接分发 |

**返回值**
无（构造函数）。若监听地址无效，抛出 `std::system_error`。

**调用（向下）**
- `net::ip::make_address()` — 解析主机地址字符串
- `net::ip::address_v4::loopback()` / `net::ip::address_v4::any()` — 特殊地址处理（localhost / 0.0.0.0）
- `acceptor_.open()` / `acceptor_.set_option()` / `acceptor_.bind()` / `acceptor_.listen()` — 配置并绑定接受器

**被调用（向上）**
- `main()` — 主线程构造 `agent::front::listener service_listener(config_ref, dispatcher)`

**涉及的知识域**
- [[ref/network/tcp|TCP 套接字]] — acceptor 绑定、SO_REUSEADDR、缓冲区设置
- [[ref/programming/boost-asio|Boost.Asio]] — io_context、acceptor、socket 选项
- [[agent/config|代理配置]] — 监听地址与缓冲区配置解析

---

### listen()

**功能说明**
启动异步接受循环协程并阻塞运行 IO 事件循环，处理所有入站连接接受事件。

**签名**
```cpp
void listen();
```

**参数**
无。

**返回值**
无返回值。该方法阻塞运行 `ioc_.run()` 直到事件循环停止。

**调用（向下）**
- `accept_loop()` — 生成异步接受循环协程
- `net::co_spawn(ioc_, accept_loop(), net::detached)` — 将协程派生到独立 IO 上下文
- `ioc_.run()` — 阻塞运行事件循环

**被调用（向上）**
- `main()` — 在独立监听线程中调用 `service_listener.listen()`

**涉及的知识域**
- [[ref/programming/boost-asio|Boost.Asio]] — co_spawn、io_context::run、detached 模式
- [[ref/programming/c++23-coroutines|C++23 协程]] — 协程派生与事件循环驱动
- [[ref/network/tcp|TCP 监听]] — accept 循环的生命周期管理

---

### make_affinity()（私有，静态）

**功能说明**
根据客户端 IP 地址和端口计算亲和性哈希值，用于负载均衡器的一致性工作线程选择。

**签名**
```cpp
[[nodiscard]] static auto make_affinity(const tcp::endpoint &endpoint) noexcept -> std::uint64_t;
```

**参数**

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `endpoint` | `const tcp::endpoint &` | 客户端端点，包含 IP 地址与端口号 |

**返回值**
`std::uint64_t` — 亲和性哈希值。IPv4 地址直接使用 32 位地址值（如 `192.168.1.1` -> `0xC0A80101`）；IPv6 地址将 16 字节折叠为 8 字节（高 8 字节 XOR 低 8 字节），保证相同客户端始终产生相同的亲和性值。

**调用（向下）**
- `endpoint.address().is_v4()` — 判断地址类型
- `endpoint.address().to_v4().to_uint()` — IPv4 地址转整数
- `endpoint.address().to_v6().to_bytes()` — IPv6 地址转字节数组

**被调用（向上）**
- `accept_loop()` — 每次接受新连接后调用 `make_affinity(remote_endpoint)` 计算亲和性值

**涉及的知识域**
- [[ref/network/tcp|TCP 连接亲和性]] — 基于客户端地址的哈希分发策略
- [[agent/front/balancer|负载均衡]] — 亲和性值用于 balancer::select() 的一致性选择
- [[agent/worker/worker|工作线程]] — 同一客户端连接倾向于分发至同一工作线程

---

### accept_loop()（私有）

**功能说明**
异步接受循环协程，持续接受入站 TCP 连接，计算亲和性值并通过负载均衡器分发至工作线程，支持反压暂停与错误指数退避。

**签名**
```cpp
auto accept_loop() -> net::awaitable<void>;
```

**参数**
无。

**返回值**
`net::awaitable<void>` — Boost.Asio 协程对象，通过 `co_spawn` 派生后由事件循环驱动执行。

**调用（向下）**
- `acceptor_.async_accept()` — 异步等待并接受新连接
- `make_affinity(remote_endpoint)` — 计算客户端亲和性哈希值
- `dispatcher_.select(affinity)` — 调用负载均衡器选择目标工作线程
- `dispatcher_.dispatch(decision.worker_index, std::move(socket))` — 将套接字移交至选中的工作线程
- `net::steady_timer::async_wait()` — 错误退避与反压暂停
- `trace::warn()` — 记录接受错误日志

**被调用（向上）**
- `listen()` — 通过 `net::co_spawn(ioc_, accept_loop(), net::detached)` 派生协程

**涉及的知识域**
- [[ref/programming/c++23-coroutines|C++23 协程]] — co_await、net::awaitable、协程挂起与恢复
- [[ref/network/tcp|TCP 连接接受]] — async_accept、TCP_NODELAY、错误处理
- [[ref/programming/boost-asio|Boost.Asio]] — 协程 executor、redirect_error、use_awaitable
- [[agent/front/balancer|负载均衡]] — select() 选择算法与 backpressure 反压机制
- [[ref/memory/arena|指数退避]] — 致命错误时的 min_delay/max_delay 退避策略

---

## 调用链

```
main()
  └─ listener::listen()
       └─ accept_loop()          [协程，事件循环驱动]
            ├─ acceptor_.async_accept()   [异步接受]
            ├─ make_affinity()            [计算亲和性]
            ├─ balancer::select()         [选择工作线程]
            └─ balancer::dispatch()       [分发套接字]
```

---

## 错误处理

| 错误场景 | 处理策略 |
|----------|----------|
| 构造时地址无效 | 抛出 `std::system_error`，附带地址解析错误信息 |
| accept 普通错误 | 记录 warn 日志，10ms 延迟后重试 |
| accept 致命错误 (fd 耗尽/内存不足) | 指数退避：从 10ms 起步，每次翻倍，上限 5120ms |
| remote_endpoint 获取失败 | 亲和性值降级为 0，不影响分发流程 |
| 全局反压触发 | 暂停 `backpressure_delay_`（默认 2ms）后继续接受 |

---

## 知识域

- [[agent/front/balancer|负载均衡器]] — 工作线程选择算法与反压机制
- [[agent/worker/worker|工作线程]] — 连接分发目标与事件循环隔离
- [[ref/network/tcp|TCP 监听与连接管理]] — acceptor 绑定、SO_REUSEADDR、套接字选项
- [[ref/programming/boost-asio|Boost.Asio 协程模型]] — co_spawn、awaitable、executor
- [[ref/programming/c++23-coroutines|C++23 协程]] — 协程语法与生命周期
