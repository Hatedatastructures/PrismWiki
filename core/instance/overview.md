---
title: Instance 模块总览
layer: core
source: "I:/code/Prism/include/prism/instance/"
module: instance
tags: [instance, overview, architecture]
updated: 2026-05-27
---

# Instance 模块总览

Instance 模块是 Prism 正向代理引擎的主体，负责前端监听、负载均衡、工作线程、会话管理和协议分发。

## 文件结构

```
include/prism/instance/
├── config.hpp          # 运行时配置
├── context.hpp         # 上下文类型
├── front/
│   ├── listener.hpp    # 监听器
│   └── balancer.hpp    # 负载均衡器
├── session/
│   └── session.hpp     # 会话生命周期
└── worker/
    ├── worker.hpp      # Worker 线程核心
    ├── launch.hpp      # 会话启动
    └── tls.hpp         # TLS 配置构建
```

## 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| [[core/instance/front/listener\|listener]] | `front/listener.hpp` | 入站连接接受 |
| [[core/instance/front/balancer\|balancer]] | `front/balancer.hpp` | Worker 选择与负载均衡 |
| [[core/instance/worker/worker\|worker]] | `worker/worker.hpp` | 线程事件循环 |
| [[core/instance/worker/launch\|launch]] | `worker/launch.hpp` | 会话创建与分发 |
| [[core/instance/worker/tls\|tls]] | `worker/tls.hpp` | SSL 上下文构建 |
| [[core/instance/session/session\|session]] | `session/session.hpp` | 连接生命周期 |

## 架构层次

```
server_context (全局共享)
    ├── config, ssl_ctx, account_store
    │
    ▼
worker_context (线程局部)
    ├── io_context, router, outbound
    │
    ▼
session_context (会话局部)
    ├── session_id, frame_arena, inbound
```

## 设计决策

### 为什么每个 worker 独立 io_context？

每个 worker 拥有独立 `io_context(1)`（单线程），避免跨线程竞争。所有 session/handler/router 在同一线程运行，无需互斥锁。只有 `dispatch_socket()` 和 `load_snapshot()` 通过 `post` 跨线程安全。

**后果**: worker 数量固定为 CPU 核心数 - 1，不支持动态调整。

### 为什么 session 用 detached co_spawn？

Session 是长时间运行的后台任务，生命周期由 `shared_ptr` 管理。`co_spawn(ioc, awaitable, detached)` 启动后无需等待完成，关闭通过状态机控制而非 awaitable 句柄。

**后果**: 异常在协程内部捕获处理，不会传播到事件循环。

## 启动流程

```
main.cpp:
  1. memory::enable_global_pooling()
  2. stealth::register_schemes()
  3. loader::load() → config
  4. trace::init()
  5. loader::build_dir() → account::directory

Instance 启动:
  6. 创建 worker 线程池 (CPU - 1)
  7. 构建 balancer（持有 worker 引用列表）
  8. 启动 listener（监听端口 → 分发到 balancer）
  9. 启动所有线程
```

## 引用关系

### 依赖

- [[core/context/context|context]]：server/worker/session 上下文
- [[core/memory/overview|memory]]：PMR 容器
- [[core/resolve/router|resolve]]：DNS 路由器
- [[core/recognition/recognition|recognition]]：协议识别
- [[core/pipeline/overview|pipeline]]：协议处理器
- [[core/stealth/overview|stealth]]：伪装配置
- [[core/outbound/overview|outbound]]：出站代理
- [[core/stats/overview|stats]]：运行时统计

### 被引用

- [[core/loader/load|loader]]：配置加载后构建 instance
