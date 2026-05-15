---
title: "worker.hpp — Worker 线程核心实现"
source: "include/prism/agent/worker/worker.hpp"
module: "agent"
type: api
tags: [agent, worker, 工作线程, 事件循环, 负载均衡]
created: 2026-05-15
updated: 2026-05-15
related:
  - agent/worker/launch
  - agent/worker/stats
  - agent/worker/tls
  - agent/front/balancer
  - agent/context
  - resolve/router
  - channel/connection/pool
  - outbound/direct
---

# worker.hpp

> 源码: `include/prism/agent/worker/worker.hpp` + `src/prism/agent/worker/worker.cpp`
> 模块: [[agent|Agent]] / worker

## 概述

Worker 线程核心实现。每个 worker 拥有独立的 `io_context` 事件循环、连接池、路由表和统计状态。worker 从主线程接收分发过来的 socket，创建会话并处理数据转发。通过负载快照向负载均衡器报告当前负载情况。

构造时根据配置初始化所有组件，包括解析反向代理路由规则、设置正向代理端点、创建 TLS 上下文等。运行时通过 `dispatch_socket` 接收主线程分发的连接，在本地事件循环中创建会话并处理数据转发。内部启动延迟监控协程，持续采集事件循环负载情况。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/front/balancer|balancer]] | 负载快照类型 `worker_load_snapshot` |
| 依赖 | [[resolve/router|router]] | DNS 路由器，决定请求转发目标 |
| 依赖 | [[agent/context|context]] | server_context / worker_context 定义 |
| 依赖 | [[agent/config|config]] | 顶层 `psm::config` 的 agent 字段 |
| 依赖 | [[agent/worker/stats|stats]] | 负载统计，记录活跃会话数和延迟 |
| 依赖 | [[agent/worker/tls|tls]] | TLS 上下文初始化 |
| 依赖 | [[channel/connection/pool|pool]] | 连接池，管理到后端的连接复用 |
| 依赖 | [[outbound/direct|direct]] | 直连出站代理 |
| 依赖 | [[agent/worker/launch|launch]] | 会话启动与连接分发 |
| 被依赖 | main | 主线程创建 worker |

## 命名空间

`psm::agent::worker`

---

## 类: worker

### 概述
代理服务工作线程核心类，封装了事件循环、连接池、路由表、TLS 上下文和统计状态等完整资源。

### 设计意图
- 每个 worker 在独立线程中创建和运行，实现"每线程一个事件循环"的无锁模型
- `ioc_(1)` 的 hint 参数表示事件循环只跑在一个线程上
- `dispatch_socket` 和 `load_snapshot` 是仅有的两个线程安全方法
- 构造时解析配置中的反向代理路由和正向代理端点

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `net::io_context` | `ioc_` | 事件循环上下文，单线程运行（hint=1） |
| `connection_pool` | `pool_` | 连接池，管理到后端的连接复用 |
| `resolve::router` | `router_` | 路由表，决定请求转发目标 |
| `std::shared_ptr<ssl::context>` | `ssl_ctx_` | TLS 上下文，为空表示明文模式 |
| `std::unique_ptr<outbound::direct>` | `outbound_direct_` | 直连出站代理 |
| `stats::state` | `metrics_` | 统计状态，记录负载指标 |
| `server_context` | `server_ctx_` | 服务端全局上下文，包含配置和共享资源 |
| `worker_context` | `worker_ctx_` | worker 线程局部上下文，包含事件循环和内存池 |

### 生命周期
- 在独立线程中创建和运行
- `run()` 方法阻塞直到事件循环停止
- 不可跨线程共享，除 `dispatch_socket` 和 `load_snapshot` 外

---

### worker::worker()

#### 功能说明
构造 worker 实例，根据配置初始化所有核心组件。首先创建 `io_context(1)` 作为事件循环引擎（hint=1 表示单线程运行），然后初始化连接池和路由表，接着根据证书配置创建 TLS 上下文，最后组装服务端和 worker 上下文对象。

构造过程中执行两项配置解析：
1. **反向代理路由**：遍历 `config.agent.reverse_map`，将虚拟域名映射到实际后端端点，无效配置记录警告日志并跳过
2. **正向代理上游**：如果配置了 `agent.positive`，设置路由器的默认上游代理端点（级联代理模式）

#### 签名
```cpp
explicit worker(const psm::config &cfg, std::shared_ptr<account::directory> account_store);
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `cfg` | `const psm::config &` | 代理服务配置，包含路由规则和资源限制 |
| `account_store` | `std::shared_ptr<account::directory>` | 账户注册表，用于认证功能 |

#### 返回值
无（构造函数）。

#### 调用（向下）
- `memory::system::thread_local_pool()` — 获取线程本地内存池
- `connection_pool` 构造函数 — 初始化连接池
- `resolve::router` 构造函数 — 初始化路由表
- `tls::make()` — 创建 TLS 上下文
- `outbound::direct` 构造函数 — 创建直连出站代理
- `router_.add_reverse_route()` — 注册反向代理路由规则
- `router_.set_positive_endpoint()` — 设置正向代理上游端点

#### 被调用（向上）
- `main()` — 主线程创建 worker 实例

#### 知识域
- [[agent/config|配置体系]] — 从 `psm::config` 解析路由规则和连接池参数
- [[resolve/router|DNS 路由]] — 反向代理路由和正向代理端点的注册
- [[agent/worker/tls|TLS 初始化]] — 根据证书配置创建 SSL 上下文
- [[channel/connection/pool|连接池]] — 连接复用的基础设施

---

### worker::run()

#### 功能说明
启动 worker 事件循环。首先启动连接池的生命周期管理，然后派生延迟监控协程持续采集事件循环负载指标（活跃会话数、事件循环延迟），最后阻塞运行 `io_context::run()` 驱动所有异步操作。该方法应在 worker 专用线程中调用，会一直阻塞直到事件循环停止。

#### 签名
```cpp
void run();
```

#### 参数
无。

#### 返回值
无。

#### 调用（向下）
- `pool_.start()` — 启动连接池生命周期管理
- `metrics_.observe(ioc_)` — 启动延迟监控协程（EMA 平滑算法，250ms 采样周期）
- `net::co_spawn(ioc_, ..., net::detached)` — 在事件循环上启动独立协程
- `ioc_.run()` — 阻塞运行事件循环

#### 被调用（向上）
- `main()` — 在 worker 专用线程中调用

#### 知识域
- [[ref/programming/boost-asio|Boost.Asio 事件循环]] — `io_context::run()` 驱动异步操作
- [[agent/worker/stats|负载统计]] — `observe()` 协程持续采集延迟指标
- [[channel/connection/pool|连接池]] — `start()` 初始化连接生命周期管理

---

### worker::dispatch_socket()

#### 功能说明
将 socket 分发到 worker 事件循环。该方法是**线程安全**的，可从主线程或其他 worker 线程调用。将已连接的 socket 投递到本 worker 的 `io_context` 中异步处理。实际处理逻辑由 `launch::dispatch` 完成，包括 socket executor 迁移、参数预配置和会话创建。

#### 签名
```cpp
void dispatch_socket(tcp::socket socket);
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `socket` | `tcp::socket` | 已连接的 TCP socket，将被移动到分发任务中 |

#### 返回值
无。

#### 调用（向下）
- `launch::dispatch()` — 跨线程连接分发（executor 迁移 + prime + start）

#### 被调用（向上）
- [[agent/front/balancer|balancer::dispatch]] — 负载均衡器选出 worker 后调用

#### 知识域
- [[agent/front/balancer|负载均衡]] — balancer 根据负载快照选择目标 worker
- [[agent/worker/launch|会话启动]] — launch::dispatch 完成实际的 socket 迁移和会话创建
- [[ref/programming/boost-asio|Boost.Asio 跨线程投递]] — `net::post()` 实现线程安全的任务投递

---

### worker::load_snapshot()

#### 功能说明
获取当前负载快照。该方法是**线程安全**的，供负载均衡器查询当前 worker 的负载情况。返回包含活跃会话数、待处理连接数和事件循环延迟的快照结构体，这些指标是负载均衡器判断是否过载、是否需要全局背压的依据。

#### 签名
```cpp
[[nodiscard]] auto load_snapshot() const noexcept -> front::worker_load_snapshot;
```

#### 参数
无。

#### 返回值
| 类型 | 说明 |
|------|------|
| `front::worker_load_snapshot` | 负载快照，包含活跃会话数、待处理连接数和延迟 |

#### 调用（向下）
- `metrics_.snapshot()` — 原子读取三项负载指标

#### 被调用（向上）
- [[agent/front/balancer|balancer]] — 负载均衡决策时查询各 worker 的负载

#### 知识域
- [[agent/worker/stats|负载统计]] — `snapshot()` 原子读取三项指标
- [[agent/front/balancer|负载均衡]] — 快照数据驱动调度决策

---

## 调用链总览

```
main → worker::run() → pool_.start() → metrics_.observe() → ioc_.run()

[[agent/front/balancer|balancer::dispatch]] → worker::dispatch_socket()
    → [[agent/worker/launch|launch::dispatch]] → launch::prime() → launch::start()
        → [[agent/session/session|make_session]] → session::start()
            → session::diversion() → [[recognition/recognition|recognition::recognize]]
                → [[agent/dispatch/table|dispatch::dispatch]] → protocol handler
```

## 知识域

- [[agent/worker/stats|负载统计]] — 活跃会话数、待处理连接数、EMA 平滑延迟三项指标
- [[agent/front/balancer|负载均衡]] — 基于快照的动态调度
- [[ref/programming/boost-asio|Boost.Asio 线程模型]] — 每线程一个 io_context 的无锁设计
- [[resolve/router|DNS 路由]] — 反向代理路由和正向代理端点
- [[channel/connection/pool|连接池]] — 到后端的连接复用
- [[agent/worker/launch|会话启动]] — socket 迁移、预配置和会话创建
