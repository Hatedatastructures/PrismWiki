---
title: "context — 三层上下文类型定义"
layer: core
source: "include/prism/context/context.hpp"
created: 2026-05-17
updated: 2026-05-28
tags: [context, server_context, worker_context, session_context, PMR]
---

# context — 三层上下文类型定义

> 源码: `include/prism/context/context.hpp`
> 聚合头文件: `include/prism/context.hpp`

## 模块职责

声明运行时上下文结构，贯穿整个请求处理生命周期。采用纯前向声明，不依赖任何实现模块头文件，确保零循环依赖。

| 层级 | 结构体 | 作用域 | 生命周期 |
|------|--------|--------|----------|
| 全局 | `context::server` | 所有线程共享 | 服务器启动到关闭 |
| 线程 | `context::worker` | 单 worker 线程 | worker 创建到销毁 |
| 连接 | `context::session` | 单个连接 | session 创建到析构 |

## 依赖关系

| 被调用者 | 用途 |
|----------|------|
| `psm::config` | server.cfg 持有的配置类型 |
| `resolve::router` | worker_ctx.router 引用 |
| `account::directory` | server.account_store 持有 |
| `account::entry` / `account::lease` | session.account_lease 类型 |
| `transport::shared_transmission` | session.inbound/outbound 类型 |
| `memory::pool` | worker_ctx.memory_pool / session.frame_arena |
| `outbound::proxy` | worker_ctx.outbound / session.outbound_proxy |

| 调用者 | 用途 |
|--------|------|
| [[worker/worker|worker.hpp]] | 持有 server + worker 上下文 |
| [[worker/launch|launch.hpp]] | 构造 session_opts 传递上下文 |
| [[session/session|session.hpp]] | 持有 session 上下文 |

---

## server 上下文

聚合服务器级别的共享资源，启动时创建，所有 worker 共享。

| 类型 | 成员 | 说明 |
|------|------|------|
| `atomic<shared_ptr<const config>>` | `cfg` | 配置对象，原子交换支持热加载 |
| `shared_ptr<ssl::context>` | `ssl_ctx` | SSL 上下文，TLS 连接复用 |
| `shared_ptr<account::directory>` | `account_store` | 账户注册表，认证管理 |

| 方法 | 签名 | 说明 |
|------|------|------|
| `config()` | `-> const psm::config &` | 无锁读取 (`cfg.load()`)，返回常量引用 |
| `swap_config()` | `void(shared_ptr<const config>)` | 原子交换配置，支持热加载 |

---

## worker 上下文

封装单个工作线程的独立资源，线程间零共享、零锁竞争。

| 类型 | 成员 | 说明 |
|------|------|------|
| `net::io_context &` | `io_context` | 事件循环引用，生命周期由 worker 管理 |
| `connect::router &` | `router` | 路由器引用，线程独享 |
| `memory::resource_pointer` | `memory_pool` | PMR 内存池指针，热路径分配 |
| `outbound::proxy *` | `outbound` | 出站代理指针（非拥有，由 worker 拥有） |
| `stats::traffic::traffic_state *` | `traffic` | 流量统计状态指针（非拥有） |

### 为什么全部使用引用和指针

**后果**: worker 上下文本身不拥有任何资源，仅作为访问入口。资源所有权由 worker 类统一管理。

**约束**:
- type: 生命周期 — server 和 worker 引用的生命周期必须长于使用它们的 session
- rule: 所有成员在同一线程内访问，无需原子操作
- violation: 跨线程访问 worker 上下文成员将导致数据竞争

---

## session_opts 构造参数

收敛 session 构造函数的参数，避免过长的参数列表（编码规范 Rule 1）。

| 类型 | 成员 | 说明 |
|------|------|------|
| `uint64_t` | `session_id` | 会话唯一标识符 |
| `server &` | `server_ctx` | 服务器上下文引用 |
| `worker &` | `worker_ctx` | 工作线程上下文引用 |
| `memory::frame_arena &` | `arena` | 帧内存池引用 |
| `function<bool(string_view)>` | `verifier` | 凭据验证回调 |
| `uint32_t` | `buffer_size` | 数据传输缓冲区大小 |
| `shared_transmission` | `inbound` | 入站传输对象（所有权转移） |

---

## session 上下文

聚合单个连接会话所需的所有资源和状态，是请求处理流程的核心数据结构。

| 类型 | 成员 | 说明 |
|------|------|------|
| `uint64_t` | `session_id` | 全局唯一标识符，递增分配 |
| `server &` | `server_ctx` | 服务器上下文常量引用 |
| `worker &` | `worker_ctx` | 工作线程上下文引用 |
| `memory::frame_arena &` | `frame_arena` | 帧内存池，会话结束回收 |
| `function<bool(string_view)>` | `credential_verifier` | 凭据验证回调（SOCKS5/HTTP 认证） |
| `account::directory *` | `account_directory` | 账户注册表指针（连接配额） |
| `uint32_t` | `buffer_size` | 数据传输缓冲区大小 |
| `shared_transmission` | `inbound` | 入站传输对象（客户端连接） |
| `shared_transmission` | `outbound` | 出站传输对象（上游连接） |
| `outbound::proxy *` | `outbound_proxy` | 出站代理指针（由 worker 设置） |
| `protocol::protocol_type` | `detected_protocol` | 识别出的协议类型 |
| `account::lease` | `account_lease` | 账户连接租约（配额管理） |
| `function<void()>` | `stream_cancel` | 多路复用流取消回调 |
| `function<void()>` | `stream_close` | 多路复用流关闭回调 |

### 为什么禁止拷贝

引用成员（server_ctx、worker_ctx、frame_arena）不可拷贝。允许移动构造（支持从 session_opts 转移），禁止移动赋值（避免已初始化对象被覆盖）。

---

## 设计要点

### 原子配置交换

server 使用 `atomic<shared_ptr<const config>>` 持有配置：`cfg.load()` 无锁读取，`cfg.store()` 原子交换，支持配置热加载。

### 线程隔离策略

worker 上下文成员全为引用/指针：io_context 单线程运行无锁竞争，router 线程独享避免 DNS 缓存跨线程同步，memory_pool 线程局部避免池竞争。

### 会话资源回收

session 禁止拷贝，移动构造支持资源转移，会话结束时 frame_arena 自动回收内存到 PMR 池。

## 相关文档

- [[config|配置类型定义]] — server.cfg 持有的类型
- [[worker/worker|Worker]] — 持有 server + worker 上下文
- [[session/session|会话]] — 持有 session 上下文
- [[account/directory|账户目录]] — server.account_store 类型
- [[core/resolve/router|DNS 路由器]] — worker_ctx.router 类型
- [[core/transport/transmission|传输层]] — session.inbound/outbound 类型
- [[core/memory/pool|PMR 内存池]] — frame_arena 类型
