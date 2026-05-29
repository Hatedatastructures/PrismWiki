---
layer: core
module: context
source: include/prism/context/context.hpp
title: Context 三层上下文
tags:
  - context
  - server
  - worker
  - session
  - lifecycle
  - zero-dependency
created: 2026-05-23
updated: 2026-05-27
---

# Context 三层上下文

三层上下文结构，按生命周期从长到短分为 `server`（全局共享）、`worker`（工作线程）、`session`（会话级）。使用纯前向声明，零循环依赖。

## 三层架构

```
┌──────────────────────────────────────────────────────────┐
│                     server (全局)                         │
│  ┌──────────────────────────────────────────────────┐    │
│  │ config (atomic<shared_ptr<const config>>)        │    │
│  │ ssl_ctx (shared_ptr<ssl::context>)               │    │
│  │ account_store (shared_ptr<account::directory>)   │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐     │
│  │              worker (每线程一个)                  │     │
│  │  io_context &   router &   memory_pool           │     │
│  │  outbound *     traffic *                         │     │
│  └─────────────────────────────────────────────────┘     │
│         × N (CPU 核心数 - 1)                              │
│                                                          │
│  ┌─────────────────────────────────────────────────┐     │
│  │              session (每连接一个)                 │     │
│  │  session_id  server_ctx &  worker_ctx &           │     │
│  │  frame_arena &  inbound  outbound                 │     │
│  │  account_lease  detected_protocol  ...            │     │
│  └─────────────────────────────────────────────────┘     │
│         × N (并发连接数)                                   │
└──────────────────────────────────────────────────────────┘
```

## server 结构体

服务器级别的共享资源，启动时创建，所有工作线程共享。

| 字段 | 类型 | 说明 |
|------|------|------|
| `cfg` | `atomic<shared_ptr<const config>>` | 配置，支持原子交换热加载 |
| `ssl_ctx` | `shared_ptr<ssl::context>` | TLS 上下文，所有 TLS 连接共享 |
| `account_store` | `shared_ptr<account::directory>` | [[core/account/directory|账户目录]] |

| 方法 | 说明 |
|------|------|
| `config()` | 无锁读取当前配置（`cfg.load()` + 解引用） |
| `swap_config(new_cfg)` | 原子交换配置（热加载） |

## worker 结构体

单个工作线程的独立资源，线程间隔离避免锁竞争。

| 字段 | 类型 | 说明 |
|------|------|------|
| `io_context` | `net::io_context &` | 事件循环核心 |
| `router` | `connect::router &` | 上游连接路由 |
| `memory_pool` | `memory::resource_pointer` | 线程局部内存池 |
| `outbound` | `outbound::proxy *` | 出站代理（可为 nullptr） |
| `traffic` | `stats::traffic::traffic_state *` | 流量统计状态（可为 nullptr） |

## session_opts 结构体

收敛 session 构造参数（遵循函数参数 ≤ 3 规范）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `session_id` | `uint64_t` | 会话唯一标识符 |
| `server_ctx` | `server &` | 服务器上下文引用 |
| `worker_ctx` | `worker &` | 工作线程上下文引用 |
| `arena` | `frame_arena &` | 帧内存池引用 |
| `verifier` | `function<bool(string_view)>` | 凭据验证函数 |
| `buffer_size` | `uint32_t` | 数据传输缓冲区大小 |
| `inbound` | `shared_transmission` | 入站传输对象 |

## session 结构体

单个连接会话的所有资源和状态，请求处理的核心数据结构。通过 `session_opts` 构造。禁止拷贝，支持移动。

| 分组 | 字段 | 说明 |
|------|------|------|
| 身份 | `session_id`, `server_ctx`, `worker_ctx`, `frame_arena` | 上下文链 |
| 认证 | `credential_verifier`, `account_directory`, `account_lease` | 账户租约 |
| 传输 | `buffer_size`, `inbound`, `outbound`, `outbound_proxy` | 双向传输 |
| 协议 | `detected_protocol` | 识别结果 |
| 多路复用 | `stream_cancel`, `stream_close` | 活跃流控制回调 |

## 设计决策

### 为什么用前向声明而非 include？

`context.hpp` 是被几乎所有模块 include 的头文件。如果它 include 了 `router.hpp`、`config.hpp` 等实现头文件，修改任何实现模块都会导致全项目重编译。通过前向声明，context.hpp 的修改面最小化。

**后果**: 仅能使用引用和指针类型，不能内联调用方法。实际 include 仅限 `entry.hpp`（需要完整定义用于 `account::lease` 成员）、`protocol/types.hpp`（枚举值）、`transport/transmission.hpp`（`shared_transmission`）。

### 为什么 server/config() 不加锁？

配置使用 `atomic<shared_ptr<const config>>`，读取端 `cfg.load()` 是无锁原子操作。进行中的请求持有旧配置的 `shared_ptr` 引用，`swap_config()` 后旧配置不会立即销毁（引用计数保护）。实现读写无竞争。

**后果**: 配置热加载后，进行中的请求仍使用旧配置直到请求完成。

### 为什么 session 用 session_opts 构造而非多参数？

遵循项目规范（Rule 1：函数参数 ≤ 3）。session 原本需要 7+ 个参数，用 `session_opts` 结构体收敛后构造函数只有一个参数。

**后果**: `session_opts` 的字段生命周期必须覆盖 session 构造完成。

## 约束

### 引用成员生命周期

**类型**: 生命周期

**规则**: `session` 的 `server_ctx`、`worker_ctx`、`frame_arena` 是引用成员，对应对象必须比 session 活得更长。

**违反后果**: 悬挂引用，未定义行为。

**源码依据**: `context.hpp:166-169`

### worker 不可跨线程

**类型**: 线程安全

**规则**: `worker` 的 `io_context`、`router`、`memory_pool` 都是线程局部资源，worker 实例只能在创建它的线程中使用。

**违反后果**: 数据竞争。

**源码依据**: `context.hpp:117-124`

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 引用 | [[core/account/overview|Account]] | account::directory, account::lease |
| 引用 | [[core/memory/overview|Memory]] | memory_pool, frame_arena |
| 引用 | [[core/protocol/overview|Protocol]] | protocol_type 枚举 |
| 引用 | [[core/outbound/overview|Outbound]] | outbound::proxy |
| 引用 | [[core/connect/overview|Connect]] | connect::router |
| 被依赖 | [[core/recognition/overview|Recognition]] | 识别结果写入 session |
| 被依赖 | [[core/pipeline/overview|Pipeline]] | 协议处理读取 session |
| 被依赖 | [[core/stealth/overview|Stealth]] | handshake_context 持有 session 指针 |

## 相关文档

- [[core/account/overview|Account 模块]] - 账户目录与租约管理
- [[core/memory/overview|Memory 模块]] - PMR 内存池与 frame_arena
- [[core/protocol/overview|Protocol 模块]] - 协议类型定义
- [[core/stealth/overview|Stealth 模块]] - handshake_context 中的 session 引用
- [[core/pipeline/overview|Pipeline 模块]] - 协议处理器使用 session 上下文
- [[core/recognition/overview|Recognition 模块]] - 协议识别结果写入 session
