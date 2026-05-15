---
title: "stats.hpp — Worker 负载统计模块"
source: "include/prism/agent/worker/stats.hpp"
module: "agent"
type: api
tags: [agent, worker, stats, 统计, 负载均衡]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[agent/front/balancer|负载均衡器]]"
  - "[[agent/worker/launch|会话启动模块]]"
  - "[[ref/network/tcp|TCP 传输]]"
  - "[[ref/programming/boost-asio|Boost.Asio 协程]]"
---

# stats.hpp

> 源码: `include/prism/agent/worker/stats.hpp`
> 实现: `src/prism/agent/worker/stats.cpp`
> 模块: [[agent|Agent]] / worker / stats

## 概述

提供单个 worker 线程的运行状态统计功能。统计数据包括活跃会话数、待处理连接数和事件循环延迟三项核心指标。这些指标被负载均衡器用于决策新连接应该分发到哪个 worker，实现基于实际负载的动态调度。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/front/balancer\|balancer]] | worker_load_snapshot 类型定义 |
| 被依赖 | [[agent/worker/worker\|worker]] | worker 持有 stats::state |
| 被依赖 | [[agent/worker/launch\|launch]] | 会话启动时更新统计 |

## 命名空间

`psm::agent::worker::stats`

---

## 类: state

### 概述
单个 worker 的运行负载统计状态。维护三项核心指标：活跃会话数、待处理连接数和事件循环延迟。

### 设计意图
延迟测量采用 EMA（指数移动平均）平滑算法，过滤短期抖动，提供稳定的负载评估依据。所有计数器均使用原子操作，支持无锁并发访问。活跃会话计数器使用 `shared_ptr` 包装，允许会话关闭回调仅捕获计数器而非整个 state 对象，避免生命周期延长问题。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `std::shared_ptr<std::atomic<std::uint32_t>>` | `active_sessions_` | 活跃会话数，shared_ptr 包装支持跨线程安全捕获 |
| `std::atomic<std::uint32_t>` | `pending_handoffs_` | 等待投递到 worker 的 socket 数 |
| `std::atomic<std::uint64_t>` | `event_loop_lag_us_` | 平滑后的事件循环延迟，单位微秒 |

---

### 构造函数

**功能说明**: 初始化所有计数器为零，分配共享的活跃会话计数器。

**签名**:
```cpp
state();
```

**参数**: 无

**返回值**: 无（构造函数）

**调用（向下）**: `std::make_shared<std::atomic<std::uint32_t>>(0U)`

**被调用（向上）**: [[agent/worker/worker\|worker]] 构造函数（worker 持有 `stats::state metrics_` 成员）

**涉及的知识域**: [[ref/programming/boost-asio\|Boost.Asio 协程]]

---

### session_open()

**功能说明**: 递增活跃会话计数器，表示一个新会话已开始处理。

**签名**:
```cpp
void session_open() noexcept;
```

**参数**: 无

**返回值**: 无

**调用（向下）**: `active_sessions_->fetch_add(1U, std::memory_order_relaxed)`

**被调用（向上）**: [[agent/worker/launch\|launch::start()]] 在会话创建成功后调用

**涉及的知识域**: [[ref/network/connection-pool\|连接配额管理]]

---

### session_close()

**功能说明**: 递减活跃会话计数器，表示一个会话已结束。

**签名**:
```cpp
void session_close() noexcept;
```

**参数**: 无

**返回值**: 无

**调用（向下）**: `active_sessions_->fetch_sub(1U, std::memory_order_relaxed)`

**被调用（向上）**: [[agent/worker/launch\|launch::start()]] 的会话关闭回调 `on_closed`（通过 `session_counter()` 捕获计数器直接递减）；异常路径中 launch::start() 也会调用进行回滚

**涉及的知识域**: [[ref/network/connection-pool\|连接配额管理]]

---

### handoff_push()

**功能说明**: 递增待处理连接计数器，表示连接已从主线程投递到 worker 但尚未开始处理。

**签名**:
```cpp
void handoff_push() noexcept;
```

**参数**: 无

**返回值**: 无

**调用（向下）**: `pending_handoffs_.fetch_add(1U, std::memory_order_relaxed)`

**被调用（向上）**: [[agent/worker/launch\|launch::dispatch()]] 在 socket 投递到 worker 事件循环前调用

**涉及的知识域**: [[ref/network/tcp\|TCP 传输]]

---

### handoff_pop()

**功能说明**: 递减待处理连接计数器，表示一个等待的 socket 已被 worker 消费处理。

**签名**:
```cpp
void handoff_pop() noexcept;
```

**参数**: 无

**返回值**: 无

**调用（向下）**: `pending_handoffs_.fetch_sub(1U, std::memory_order_relaxed)`

**被调用（向上）**: [[agent/worker/launch\|launch::dispatch()]] 内部 lambda 在 worker 线程中开始处理 socket 时调用

**涉及的知识域**: [[ref/network/tcp\|TCP 传输]]

---

### session_counter()

**功能说明**: 获取活跃会话计数器的共享指针，允许会话关闭回调仅捕获计数器而非整个 state 对象。

**签名**:
```cpp
[[nodiscard]] auto session_counter() const noexcept
    -> const std::shared_ptr<std::atomic<std::uint32_t>> &;
```

**参数**: 无

**返回值**: `active_sessions_` 的常量引用，调用方可通过该指针直接操作活跃会话计数

**调用（向下）**: 无（仅返回内部成员）

**被调用（向上）**: [[agent/worker/launch\|launch::start()]] 获取计数器用于构造会话关闭回调 `on_closed`

**涉及的知识域**: [[ref/programming/c++23-coroutines\|RAII 资源管理]]

---

### snapshot()

**功能说明**: 原子地读取三项指标并打包成快照结构体返回，供负载均衡器做调度决策。

**签名**:
```cpp
[[nodiscard]] auto snapshot() const noexcept
    -> front::worker_load_snapshot;
```

**参数**: 无

**返回值**: `front::worker_load_snapshot` 结构体，包含 `active_sessions`、`pending_handoffs`、`event_loop_lag_us` 三个字段

**调用（向下）**: `active_sessions_->load()`、`pending_handoffs_.load()`、`event_loop_lag_us_.load()`（均使用 `memory_order_relaxed`）

**被调用（向上）**: [[agent/worker/worker\|worker::load_snapshot()]] 转发调用，供 [[agent/front/balancer\|balancer]] 查询

**涉及的知识域**: [[ref/network/tcp\|EMA 平滑算法]]

---

### observe()

**功能说明**: 协程方式周期性采样事件循环延迟，每 250ms 测量一次调度延迟并经 EMA 平滑后存储。

**签名**:
```cpp
auto observe(net::io_context &ioc)
    -> net::awaitable<void>;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `ioc` | `net::io_context &` | 当前 worker 的 io_context，用于创建 steady_timer |

**返回值**: `net::awaitable<void>` 协程，持续运行直到事件循环停止或定时器被取消

**调用（向下）**:
- `net::steady_timer` 创建和 `async_wait` 定时触发
- `std::chrono::steady_clock::now()` 测量实际触发时间偏差
- EMA 公式: `smoothed_lag_us = (smoothed_lag_us * 7 + effective_lag) / 8`
- `event_loop_lag_us_.store()` 存储平滑结果
- `trace::debug()` 记录定时器错误

**被调用（向上）**: [[agent/worker/worker\|worker::run()]] 通过 `net::co_spawn(ioc_, metrics_.observe(ioc_), net::detached)` 启动

**涉及的知识域**:
- [[ref/programming/boost-asio\|Boost.Asio 协程]]
- [[ref/network/tcp\|EMA 平滑算法]]

### observe() 算法细节

1. **预热阶段**（前 16 个样本）：采集样本估算系统调度抖动基线 `jitter_baseline_us`，期间延迟值输出为 0
2. **正常采样**：从原始延迟中扣除基线抖动得到有效延迟
3. **噪声过滤**：低于 1ms 的有效延迟视为零
4. **上限截断**：原始延迟超过 20ms 的截断为 20ms，防止单次异常值污染
5. **EMA 平滑**：权重 7/8 给历史值、1/8 给新样本

---

## 知识域

- [[ref/programming/boost-asio\|Boost.Asio 协程]]
- [[ref/network/tcp\|EMA 平滑算法]]
- [[ref/network/connection-pool\|连接配额管理]]
