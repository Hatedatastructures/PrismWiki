---
title: Agent 模块总览
layer: core
source: "I:/code/Prism/include/prism/agent/"
created: 2026-05-17
tags: [agent, overview, architecture, config, context]
---

# Agent 模块总览

> 源码位置: `include/prism/agent/`

## 模块职责

Agent 是 Prism 正向代理引擎的主体模块，负责：

- **前端监听**: 接受入站连接，计算亲和性哈希
- **负载均衡**: 选择 worker 分发连接
- **工作线程**: 每线程独立 io_context，无锁设计
- **会话管理**: 连接生命周期，协议识别分派
- **账户管理**: 用户认证、连接配额控制
- **协议分发**: 处理器注册表，管道入口

## 文件结构

### 头文件

```
include/prism/agent/
├── agent.hpp           # 聚合头文件
├── config.hpp          # 运行时配置类型定义
├── context.hpp         # 运行时上下文类型定义
├── account/
│   ├── entry.hpp       # 账户条目
│   └── directory.hpp   # 账户目录（认证注册表）
├── dispatch/
│   └── table.hpp       # 处理器分发表
├── front/
│   ├── listener.hpp    # 监听器
│   └── balancer.hpp    # 负载均衡器
├── session/
│   └── session.hpp     # 会话生命周期管理
├── worker/
│   ├── worker.hpp      # Worker 线程核心
│   ├── launch.hpp      # 会话启动
│   ├── stats.hpp       # 统计信息
│   └── tls.hpp         # TLS 配置构建
```

### 源文件

```
src/prism/agent/
├── account/
│   └── directory.cpp   # 账户目录实现
├── front/
│   ├── listener.cpp    # 监听器实现
│   └── balancer.cpp    # 负载均衡器实现
├── session/
│   └── session.cpp     # 会话生命周期实现
├── worker/
│   ├── worker.cpp      # Worker 线程实现
│   ├── launch.cpp      # 会话启动实现
│   ├── stats.cpp       # 统计信息实现
│   └── tls.cpp         # TLS 配置构建实现
```

## 核心组件

| 组件 | 头文件 | 源文件 | 职责 |
|------|---------|---------|------|
| [[config|配置]] | `config.hpp` | header-only | 配置类型定义 |
| [[context|上下文]] | `context.hpp` | header-only | 上下文类型定义 |
| [[account/directory|账户目录]] | `account/directory.hpp` | `account/directory.cpp` | 认证注册表 |
| [[account/entry|账户条目]] | `account/entry.hpp` | header-only | 账户状态 |
| [[dispatch/table|处理器表]] | `dispatch/table.hpp` | header-only | 协议处理器分发 |
| [[front/listener|监听器]] | `front/listener.hpp` | `front/listener.cpp` | 入站连接接受 |
| [[front/balancer|负载均衡器]] | `front/balancer.hpp` | `front/balancer.cpp` | Worker 选择 |
| [[session/session|会话]] | `session/session.hpp` | `session/session.cpp` | 生命周期管理 |
| [[worker/worker|Worker]] | `worker/worker.hpp` | `worker/worker.cpp` | 线程事件循环 |
| [[worker/launch|会话启动]] | `worker/launch.hpp` | `worker/launch.cpp` | 会话创建 |
| [[worker/stats|统计]] | `worker/stats.hpp` | `worker/stats.cpp` | 负载指标 |
| [[worker/tls|TLS配置]] | `worker/tls.hpp` | `worker/tls.cpp` | SSL 上下文构建 |

## 架构层次

```
server_context (全局共享)
    │
    ├── config (配置引用)
    ├── ssl_ctx (SSL 上下文)
    └── account_store (账户注册表)
    │
    ▼
worker_context (线程局部)
    │
    ├── io_context (事件循环)
    ├── router (DNS 路由器)
    ├── memory_pool (PMR 内存池)
    └── outbound (出站代理)
    │
    ▼
session_context (会话局部)
    │
    ├── session_id (唯一标识)
    ├── frame_arena (帧内存池)
    ├── inbound (入站传输)
    ├── outbound (出站传输)
    └── credential_verifier (验证回调)
```

## 启动流程

Agent 模块在启动流程中的位置：

```
main.cpp 启动流程:
    │
    ├── 1. memory::system::enable_global_pooling()
    ├── 2. stealth::register_all_schemes()
    ├── 3. loader::load() → psm::config
    ├── 4. trace::init()
    ├── 5. loader::build_account_directory() → account::directory
    │
    ▼
Agent 启动阶段:
    │
    ├── 6. 创建 worker 线程池 (CPU 核心 - 1)
    │       └── worker 构造 server_context + worker_context
    │
    ├── 7. 构建 balancer
    │       └── 持有 worker 引用列表
    │
    ├── 8. 启动 listener
    │       └── 监听 addressable 端点
    │       └── 分发到 balancer
    │
    └── 9. 启动所有线程
            ├── worker 线程运行 io_context.run()
            └── listener 线程接受连接
```

## 协议处理流程

```
listener 接受连接
    │
    ├── 计算亲和性哈希 (socket.remote_endpoint)
    │
    ▼
balancer 选择 worker
    │
    ├── 查询各 worker 负载快照
    ├── 选择最小负载 worker
    │
    ▼
worker::dispatch_socket()
    │
    ├── 投递到 io_context
    │
    ▼
launch::dispatch()
    │
    ├── 创建 session_params
    ├── 调用 make_session()
    │
    ▼
session::start()
    │
    ├── 构造 session_context
    ├── 启动 diversion() 协程
    │
    ▼
session::diversion()
    │
    ├── 预读 24 字节
    ├── detect_from_transmission() → protocol_type
    │
    ├── dispatch::table 获取 handler
    │
    ▼
handler 处理
    │
    ├── 调用 pipeline 协议处理器
    ├── 通过 router 建立上游连接
    │
    ▼
tunnel 双向转发
    │
    ├── inbound → outbound (请求)
    ├── outbound → inbound (响应)
    │
    ▼
会话结束 → session::close()
```

## 设计要点

### 无锁设计

每个 worker 拥有独立 io_context（单线程），避免跨线程竞争：

- `ioc_(1)` 构造参数确保单线程事件循环
- 所有 session/handler/router 在同一线程运行
- 只有 `dispatch_socket()` 和 `load_snapshot()` 跨线程安全

### PMR 内存策略

热路径零堆分配：

- 配置使用 `memory::string/memory::vector/memory::map`
- session 使用 `frame_arena` 帧内存池
- router/channel 使用 `thread_local_pool`

### 协程纯度

禁止阻塞操作：

- 所有异步操作返回 `net::awaitable<T>`
- 使用 `co_await` 执行顺序异步操作
- 使用 `net::co_spawn` 启动独立协程

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/overview|Memory]] | PMR 容器、内存池 |
| 依赖 | [[resolve/router|Resolve]] | DNS 路由器 |
| 依赖 | [[channel/overview|Channel]] | 连接池、传输层 |
| 依赖 | [[protocol/overview|Protocol]] | 协议配置 |
| 依赖 | [[stealth/overview|Stealth]] | 伪装配置 |
| 依赖 | [[pipeline/overview|Pipeline]] | 协议处理器 |
| 依赖 | [[recognition/recognition|Recognition]] | 协议识别 |
| 依赖 | [[outbound/proxy|Outbound]] | 出站代理 |
| 被依赖 | [[loader/load|Loader]] | 配置加载后构建 Agent |
| 被依赖 | [[core/startup|启动流程]] | Agent 是启动核心 |

## 配置映射

Agent 配置项与 `configuration.json` 映射：

```json
{
  "agent": {
    "limit": { "blacklist": true },
    "positive": { "host": "", "port": 0 },
    "addressable": { "host": "0.0.0.0", "port": 443 },
    "certificate": { "key": "path", "cert": "path" },
    "authentication": { "users": [...] },
    "camouflage": "/hidden-path",
    "reverse_map": {}
  }
}
```

详见 [[config|配置详解]]

## 知识域

- [[config|配置类型定义]] — endpoint, limit, certificate, authentication
- [[context|上下文类型定义]] — server_context, worker_context, session_context
- [[core/architecture|六阶段流水线架构]]
- [[core/startup|启动流程详解]]
- [[core/flow|协议处理流程详解]]