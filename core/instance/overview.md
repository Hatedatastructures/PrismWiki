---
title: Instance 模块总览
layer: core
source: "include/prism/instance/"
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


## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| worker 数量等于 CPU 核心数 - 1 | 每个 worker 独占一个 io_context 线程 | worker 数量与核心数不匹配导致性能退化 | `instance/config.hpp` |
| TLS 证书路径必须有效 | PEM 格式证书文件必须存在 | 启动失败 | `instance/config.hpp` |
| 认证用户列表为空则无认证 | 空 auth.users 表示接受所有连接 | 安全性降低 | `instance/config.hpp` |
| limits 连接限制硬性上限 | 超过 max_connections 拒绝新连接 | 客户端连接被拒 | `instance/config.hpp` |

## 故障场景

### 1. TLS 证书加载失败

**触发条件**: 配置的证书文件路径不存在或格式错误

**传播路径**: worker 初始化 -> tls::configure() 失败 -> 启动异常退出

**外部表现**: 服务无法启动

### 2. 监听端口被占用

**触发条件**: 配置的监听端口已被其他进程占用

**传播路径**: listener::start() -> bind 失败 -> 异常退出

**外部表现**: 服务无法启动，日志显示地址已使用

### 3. 所有 worker 线程饱和

**触发条件**: 并发连接数超过 limits 配置

**传播路径**: balancer 分发 -> 所有 worker active_sessions 达到上限 -> 拒绝新连接

**外部表现**: 新连接被关闭

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| instance -> [[core/recognition/overview\|recognition]] | 调用 | launch 中 session 调用 recognition::recognize() 识别协议 |
| instance -> [[core/protocol/overview\|protocol]] | 调用 | session::diversion() 分发到协议处理器 |
| instance -> [[core/connect/overview\|connect]] | 调用 | 协议处理器通过 connect::dial 建立上游连接 |
| instance -> [[core/stealth/overview\|stealth]] | 调用 | recognition 识别为 TLS 后调用 stealth 方案执行握手 |
| instance <- [[core/stats/overview\|stats]] | 被依赖 | worker 持有 worker_load 和 traffic_state |

## 变更敏感度

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| instance::config 字段变更 | 所有下游模块 | 序列化/反序列化需同步更新 |
| worker 线程模型变更 | stats/balancer | 负载均衡和统计逻辑需适配 |
| TLS 配置变更 | 所有 TLS 连接 | 证书、密码套件影响客户端兼容性 |

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
