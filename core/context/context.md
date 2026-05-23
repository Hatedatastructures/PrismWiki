---
layer: core
source: include/prism/context/context.hpp
title: Context 三层上下文
module: context
tags:
  - context
  - server
  - worker
  - session
  - lifecycle
  - zero-dependency
created: 2026-05-23
updated: 2026-05-23
---

# Context 三层上下文

> 源码位置: `include/prism/context/context.hpp`

## 概述

Context 模块定义了三层上下文结构，按生命周期从长到短分为 `server`（全局共享）、`worker`（工作线程）、`session`（会话级）。使用纯前向声明，不依赖任何实现模块头文件，确保零循环依赖。

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

`server` 聚合服务器级别的共享资源，在服务器启动时创建，被所有工作线程共享。

### 定义

```cpp
namespace psm::context
{
    struct server
    {
        std::atomic<std::shared_ptr<const psm::config>> cfg;    // 配置（可原子交换）
        std::shared_ptr<ssl::context> ssl_ctx;                  // SSL 上下文
        std::shared_ptr<account::directory> account_store;      // 账户注册表

        [[nodiscard]] auto config() const -> const psm::config &;
        void swap_config(std::shared_ptr<const psm::config> new_cfg);
    };
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `cfg` | `atomic<shared_ptr<const config>>` | 服务器配置，支持原子交换实现热加载 |
| `ssl_ctx` | `shared_ptr<ssl::context>` | TLS 上下文，所有 TLS 连接共享 |
| `account_store` | `shared_ptr<account::directory>` | [[core/account/directory|账户目录]]，全局共享 |

### 方法

#### config()

```cpp
[[nodiscard]] auto config() const -> const psm::config &;
```

获取当前配置（无锁读取）。通过 `atomic load` 加载 `shared_ptr`，解引用返回配置常量引用。

#### swap_config()

```cpp
void swap_config(std::shared_ptr<const psm::config> new_cfg);
```

原子交换配置，用于配置热加载。调用后所有新请求立即使用新配置，进行中的请求仍使用旧配置（通过已持有的引用）。

### 配置热加载流程

```
1. 接收到配置重载信号
    │
    ▼
2. 解析新配置文件 → shared_ptr<const config>
    │
    ▼
3. server.swap_config(new_cfg)
    │   └── atomic store 替换 shared_ptr
    │
    ▼
4. 后续请求通过 config() 获取新配置
    │
    └── 进行中的请求仍持有旧配置引用 → 安全
```

## worker 结构体

`worker` 封装单个工作线程的独立资源，实现线程间资源隔离和避免锁竞争。

### 定义

```cpp
namespace psm::context
{
    struct worker
    {
        net::io_context &io_context;           // I/O 上下文引用
        connect::router &router;               // 路由器引用
        memory::resource_pointer memory_pool;  // 内存池资源指针
        outbound::proxy *outbound{nullptr};    // 出站代理指针（由 worker 拥有）
        stats::traffic::traffic_state *traffic{nullptr}; // 流量统计状态指针
    };
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `io_context` | `net::io_context &` | Boost.Asio I/O 上下文引用，事件循环核心 |
| `router` | `connect::router &` | 路由器引用，管理上游连接 |
| `memory_pool` | `memory::resource_pointer` | 线程局部内存池，无锁分配 |
| `outbound` | `outbound::proxy *` | 出站代理（正向代理），可为 nullptr |
| `traffic` | `stats::traffic::traffic_state *` | 流量统计状态，可为 nullptr |

### 线程隔离模型

```
Thread 1 (worker)                Thread 2 (worker)
┌───────────────────┐           ┌───────────────────┐
│ io_context_1      │           │ io_context_2      │
│ router_1          │           │ router_2          │
│ memory_pool_1     │           │ memory_pool_2     │
│ (无锁分配)        │           │ (无锁分配)        │
└────────┬──────────┘           └────────┬──────────┘
         │                               │
         └───────────┬───────────────────┘
                     │
              ┌──────▼──────┐
              │   server    │
              │ (共享资源)  │
              │ config      │
              │ ssl_ctx     │
              │ account_dir │
              └─────────────┘
```

每个 worker 拥有独立的 `io_context`、`router` 和 `memory_pool`，避免线程间竞争。共享资源通过 `server` 结构体的原子操作访问。

## session 结构体

`session` 聚合单个连接会话所需的所有资源和状态，是请求处理流程的核心数据结构。

### 定义

```cpp
namespace psm::context
{
    struct session
    {
        // 禁止拷贝
        session(const session &) = delete;
        session &operator=(const session &) = delete;
        // 支持移动
        session(session &&) = default;
        session &operator=(session &&) = delete;

        session(std::uint64_t sid, server &srv, worker &wrk,
                memory::frame_arena &arena,
                std::function<bool(std::string_view)> verifier,
                const std::uint32_t buf_size, shared_transmission in);

        // 身份与上下文
        std::uint64_t session_id{0};
        server &server_ctx;
        worker &worker_ctx;
        memory::frame_arena &frame_arena;

        // 认证
        std::function<bool(std::string_view)> credential_verifier;
        account::directory *account_directory{nullptr};
        account::lease account_lease;

        // 传输
        std::uint32_t buffer_size;
        shared_transmission inbound;
        shared_transmission outbound;
        outbound::proxy *outbound_proxy{nullptr};

        // 协议
        protocol::protocol_type detected_protocol{protocol::protocol_type::unknown};

        // 多路复用
        std::function<void()> active_stream_cancel;
        std::function<void()> active_stream_close;
    };
}
```

### 字段分组

#### 身份与上下文

| 字段 | 类型 | 说明 |
|------|------|------|
| `session_id` | `uint64_t` | 会话唯一标识符（递增分配） |
| `server_ctx` | `server &` | 服务器全局上下文引用 |
| `worker_ctx` | `worker &` | 工作线程上下文引用 |
| `frame_arena` | `frame_arena &` | 帧内存池引用（栈上单调分配器） |

#### 认证与账户

| 字段 | 类型 | 说明 |
|------|------|------|
| `credential_verifier` | `function<bool(string_view)>` | 凭据验证函数（密码/UUID 校验） |
| `account_directory` | `account::directory *` | [[core/account/directory|账户目录]]指针 |
| `account_lease` | `account::lease` | [[core/account/entry|账户连接租约]]（RAII） |

#### 传输层

| 字段 | 类型 | 说明 |
|------|------|------|
| `buffer_size` | `uint32_t` | 数据传输缓冲区大小（字节） |
| `inbound` | `shared_transmission` | 入站传输对象（客户端→服务器） |
| `outbound` | `shared_transmission` | 出站传输对象（服务器→上游） |
| `outbound_proxy` | `outbound::proxy *` | 出站代理指针 |

#### 协议识别

| 字段 | 类型 | 说明 |
|------|------|------|
| `detected_protocol` | `protocol_type` | 识别出的协议类型 |

#### 多路复用控制

| 字段 | 类型 | 说明 |
|------|------|------|
| `active_stream_cancel` | `function<void()>` | 活跃流取消回调 |
| `active_stream_close` | `function<void()>` | 活跃流关闭回调 |

## session 构造流程

```
listener 接受连接
    │
    ▼
balancer 选择 worker
    │
    ▼
launch 创建 session
    │
    ├── 分配 session_id (递增)
    ├── 传入 server & 引用
    ├── 传入 worker & 引用
    ├── 传入 frame_arena & 引用
    ├── 创建 credential_verifier (闭包捕获账户目录)
    ├── 传入 buffer_size (来自配置)
    └── 传入 inbound transport
            │
            ▼
        session 处理流程
            │
            ├── recognition::recognize()
            │       → session.detected_protocol = protocol_type
            │
            ├── dispatch::handler(detected_protocol)
            │       → 协议处理器执行
            │
            ├── try_acquire(account_directory, credential)
            │       → session.account_lease = lease
            │
            ├── 建立出站连接
            │       → session.outbound = transport
            │
            └── tunnel 双向转发
                    │
                    ├── accumulate_uplink(account_lease.get(), n)
                    └── accumulate_downlink(account_lease.get(), n)
```

## 前向声明策略

Context 头文件仅使用前向声明引用其他模块，不 include 任何实现头文件：

```cpp
// 前向声明（零实现模块依赖）
namespace psm { struct config; }
namespace psm::connect { class router; }
namespace psm::account { class directory; class lease; }
namespace psm::outbound { class proxy; }
namespace psm::stats::traffic { class traffic_state; }
namespace psm::memory { class frame_arena; using resource_pointer = std::pmr::memory_resource *; }
```

实际 include 的头文件仅包含类型定义：
- `<prism/transport/transmission.hpp>` -- `shared_transmission` 类型
- `<prism/account/entry.hpp>` -- `entry` 和 `lease` 完整定义
- `<prism/protocol/protocol_type.hpp>` -- `protocol_type` 枚举

这种设计确保了 context.hpp 不会被实现细节污染，修改任何实现模块不需要重新编译包含 context.hpp 的代码。

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
