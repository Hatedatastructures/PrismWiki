---
layer: core
source: I:/code/Prism/include/prism/instance/worker/worker.hpp
title: worker 模块
created: 2026-05-17
updated: 2026-05-28
tags: [instance, worker, io-context, thread-pool, session]
---

# worker 模块

> 源码: `include/prism/instance/worker/worker.hpp`

## 模块职责

Worker 线程核心实现。每个 worker 拥有独立的 `io_context` 事件循环、连接池、路由表和统计状态。从主线程接收分发过来的 socket，创建会话并处理数据转发，通过负载快照向负载均衡器报告当前负载情况。

## 核心接口

### worker 类

| 方法 | 线程安全 | 说明 |
|------|----------|------|
| `worker(cfg, account_store)` | 否 | 构造，初始化所有核心组件 |
| `run()` | 否 | 启动事件循环，阻塞运行直到停止 |
| `stop()` | 否 | 停止 io_context，实现优雅停机 |
| `dispatch_socket(socket)` | **是** | 将 socket 分发到 worker 事件循环 |
| `load_snapshot()` | **是** | 获取当前负载快照 |

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `ioc_` | `net::io_context` | 事件循环，单线程运行 `ioc_(1)` |
| `pool_` | `connection_pool` | 连接池，管理到后端的连接复用 |
| `router_` | `connect::router` | 路由表，决定请求转发目标 |
| `ssl_ctx_` | `shared_ptr<ssl::context>` | TLS 上下文，空表示明文模式 |
| `outbound_direct_` | `unique_ptr<outbound::direct>` | 直连出站代理 |
| `metrics_` | `stats::runtime::worker_load` | 负载统计 |
| `traffic_` | `stats::traffic::traffic_state` | 流量统计 |
| `server_ctx_` | `context::server` | 服务端全局上下文 |
| `worker_ctx_` | `context::worker` | worker 线程局部上下文 |

---

## 构造流程

1. 创建 `io_context(1)` 单线程事件循环
2. 初始化连接池 `pool_`（绑定到本 worker 的 io_context + 线程局部内存池）
3. 初始化路由表 `router_`（解析反向代理路由规则）
4. 创建 TLS 上下文 `ssl_ctx_`（根据证书配置，未配置则为空）
5. 创建直连出站代理 `outbound_direct_`
6. 组装 `server_ctx_` 和 `worker_ctx_`

---

## 事件循环机制

`run()` 启动延迟观测协程后阻塞运行 `ioc_.run()`。

**关键特性**:

| 特性 | 说明 |
|------|------|
| 单线程 | `ioc_(1)` 确保只有一个线程运行 `run()` |
| 无锁 handler | handler 在同一线程顺序执行，不需要互斥锁 |
| 自动退出 | 无 outstanding work 且队列为空时 `run()` 返回 |
| Work guard | `executor_work_guard` 防止无任务时提前退出 |

### 任务调度模型

所有连接通过 `dispatch_socket()` -> `ioc_.post()` 进入 worker 的内部 FIFO 队列。handler 顺序执行，无优先级区分。协程遇到 `co_await` 时挂起不阻塞后续任务，I/O 完成时作为新 handler 入队。

### 延迟观测

`metrics_.observe(ioc_)` 协程以 250ms 周期采样事件循环调度延迟，经 EMA 平滑后存入 `event_loop_lag_us_`，供 balancer 读取。

---

## 调度策略

`dispatch_socket` 通过 `post()` 将任务入队：

| 阶段 | 动作 |
|------|------|
| 入队 | `handoff_push()` 递增待处理计数 |
| 出队 | `handoff_pop()` 递减待处理计数，配置 socket 参数 |
| 启动 | `launch::start()` 创建会话，`session_open()` 递增活跃计数 |
| 异常 | `session_close()` 立即递减活跃计数 |

### 公平性

| 机制 | 保证 |
|------|------|
| FIFO 队列 | 先到的任务先执行 |
| 无优先级 | 所有 post 任务平等对待 |
| 无饿死 | 每个任务最终都会被执行 |

---

## 负载均衡交互

### 快照报告

`load_snapshot()` 返回 `stats::worker_snapshot`，包含 `active_sessions`、`pending_handoffs`、`event_loop_lag_us` 三个原子读取的指标。

### 指标更新时机

| 事件 | 指标变化 |
|------|----------|
| 新连接 dispatch | `pending_handoffs++` |
| dispatch 任务执行 | `pending_handoffs--` |
| 会话开始处理 | `active_sessions++` |
| 会话结束 | `active_sessions--` |
| 延迟采样 (250ms) | `event_loop_lag_us` EMA 更新 |

### 闭环

observe 协程采集延迟 -> EMA 平滑 -> balancer 读取快照 -> score() 评分 -> 过载标记/背压 -> 选择 secondary 或暂停接受连接。

---

## 线程局部资源隔离

每个 worker 拥有完全独立的资源集合，线程间零共享（除 `server_ctx_`）。

| 资源 | 隔离级别 | 说明 |
|------|----------|------|
| io_context | 线程独占 | 单线程运行，无锁 |
| connection_pool | 线程独占 | socket 绑定本 worker io_context，仅本线程复用 |
| router | 线程独占 | DNS 缓存线程局部，无跨线程同步 |
| PMR memory_pool | 线程独占 | `thread_local_pool()`，热路径无竞争 |
| ssl_ctx_ | 跨 worker 共享 | `ssl::context` 只读配置，握手后每连接独立 `ssl::stream` |

### 为什么 ssl_ctx 可以共享

`ssl::context` 是只读配置对象。TLS 握手时使用，握手后每个连接持有独立的 `ssl::stream`。共享可减少证书重复加载和内存占用。

### 资源生命周期

| 资源 | 创建 | 销毁 | 作用域 |
|------|------|------|--------|
| io_context | worker 构造 | worker 析构 | worker 生命周期 |
| connection_pool | worker 构造 | worker 析构 | worker 生命周期 |
| router | worker 构造 | worker 析构 | worker 生命周期 |
| ssl_ctx_ | 启动时 `tls::make` | 服务器关闭 | 所有 worker 共享 |
| frame_arena | session 构造 | session 析构 | 单个会话 |

## 相关文档

- [[core/agent/context|上下文模块]]
- [[core/agent/session/session|会话模块]]
- [[core/agent/worker/launch|启动模块]]
- [[core/agent/worker/stats|统计模块]]
- [[core/agent/worker/tls|TLS 模块]]
- [[core/agent/front/balancer|负载均衡器]]
- [[core/resolve/router|路由模块]]
