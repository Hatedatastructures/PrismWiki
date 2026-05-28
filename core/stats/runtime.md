---
title: Runtime 运行时统计
layer: core
source:
  - I:/code/Prism/include/prism/stats/runtime.hpp
  - I:/code/Prism/src/prism/stats/runtime.cpp
module: stats
tags: [stats, runtime, worker, load-balancing]
created: 2026-05-23
updated: 2026-05-23
---

# Runtime 运行时统计

Runtime 子模块提供两个核心组件：`worker_load` 负责单个 worker 的运行负载监控，`system_state` 负责全局运行状态追踪。`worker_load` 从原有 `instance::worker::stats::state` 迁移而来，算法零改动。

> 源码: `include/prism/stats/runtime.hpp` | 实现: `src/prism/stats/runtime.cpp`

## worker_load

单个 worker 的运行负载统计，从原有 `psm::stats::state` 迁移，接口和算法完全不变。

### 接口

| 方法 | 说明 |
|------|------|
| `session_open()` noexcept | 活跃会话 +1，在 launch::start() 中调用 |
| `session_close()` noexcept | 活跃会话 -1，在 launch::dispatch() 的 on_closed 回调中调用 |
| `handoff_push()` noexcept | 分发队列入队 +1 |
| `handoff_pop()` noexcept | 分发队列出队 -1 |
| `session_counter() -> shared_ptr<atomic<uint32_t>>` | 获取活跃会话计数器共享指针 |
| `snapshot() -> worker_snapshot` | 返回活跃会话数、待分发数、事件循环延迟 |
| `observe(io_context&) -> awaitable<void>` | 启动事件循环延迟监测协程 |

### 为什么 active_sessions_ 使用 shared_ptr 包装？

**问题**: 会话关闭回调（`on_closed`）可能在 session 对象析构后才执行。如果回调直接捕获 `worker_load*` 裸指针，存在悬挂指针风险。

**选择**: 通过 `shared_ptr<atomic<uint32_t>>` 包装计数器。关闭回调仅捕获计数器的 shared_ptr，独立于 worker 生命周期。

**后果**: 即使 worker 对象已部分析构，回调持有的 `shared_ptr` 仍保证计数器有效。

### handoff 调用时序

```
listener.accept() -> balancer.select()
                        |
                        v
                   dispatch_socket()
                    handoff_push()     <- pending = 1
                        |
                        v
                   ioc.post(task)
                        |
                        v (worker 线程取出任务)
                   handoff_pop()      <- pending = 0
                        |
                        v
                   session_open()     <- active = 1
```

### observe() 事件循环延迟监测

observe() 是在 worker 的 `io_context` 上持续运行的协程，每 250ms 采样一次实际等待时间。

#### 算法流程

```
observe 协程启动
    |
    v
[预热阶段: 16 次采样]
  1. 设置 timer 到 expected_time + 250ms
  2. co_await timer.async_wait()
  3. 测量 actual_time - expected_time 偏差
  4. 截断到 20ms 上限
  5. 增量更新 jitter_baseline_us（移动平均）
  6. event_loop_lag_us_ = 0（预热期间报告零延迟）
    |
    v
[正常采样阶段: 无限循环]
  1. 设置 timer 到 expected_time + 250ms
  2. co_await timer.async_wait()
  3. raw_lag = actual - expected
  4. capped = min(raw, 20ms)
  5. effective = capped - jitter_baseline
  6. if effective < 1ms -> effective = 0
  7. smoothed = (smoothed * 7 + effective) / 8   (EMA)
  8. lag_us_ = smoothed
```

#### 关键参数

| 参数 | 值 | 作用 |
|------|-----|------|
| 采样间隔 | 250ms | 平衡精度和开销 |
| 预热样本数 | 16 | 估算系统调度抖动基线 |
| EMA 系数 | 7/8 | 新样本权重 12.5%，历史权重 87.5% |
| 小抖动阈值 | 1ms | 低于此值视为零延迟 |
| 上限截断 | 20ms | 防止单次异常值污染 |

预热阶段计算的是"空负载下的期望调度延迟"（操作系统定时器精度和 Asio 内部队列处理时间的基线）。正常采样时扣除基线，只保留真实的事件循环积压延迟。

#### 定时器错误处理

当 `io_context` 停止时，异步操作被取消，`ec` 为 `operation_aborted`，这是正常的退出路径。其他错误码记录 debug 日志后退出。

### 数据流向

`worker_load::snapshot()` 被 [[core/instance/worker/worker|worker::load_snapshot()]] 调用，进而被 [[core/instance/front/balancer|balancer]] 的 `score()` 方法消费，用于负载均衡决策。

## system_state

全局运行状态单例，记录进程启动时间和 worker 数量。

### 接口

| 方法 | 说明 |
|------|------|
| `instance() -> system_state&` | Meyers' Singleton，线程安全的局部静态变量 |
| `mark_started(uint32_t worker_count)` noexcept | 标记系统已启动，使用 `exchange` 原子操作保证只初始化一次 |
| `snapshot() -> runtime_snapshot` | 返回运行时间（秒）和 worker 数量；未启动时返回空快照 |

**防重复调用**: `mark_started()` 使用 `exchange` 原子操作确保只初始化一次。后续调用（如配置热重载）不会重置启动时间。

**调用时机**: 在 `main.cpp` 启动流程中调用一次。

**时间源**: 使用 `steady_clock` 计算运行时间，不受 NTP 调整或手动修改系统时间影响。

内部状态: `atomic<bool> started_` + `steady_clock::time_point start_time_` + `uint32_t worker_count_`

## 线程安全

| 组件 | 方法 | 安全 | 机制 |
|------|------|------|------|
| worker_load | session_open/close | 是 | atomic fetch_add/fetch_sub |
| worker_load | handoff_push/pop | 是 | atomic fetch_add/fetch_sub |
| worker_load | snapshot() | 是 | atomic load x3 |
| worker_load | observe() | 否 | 必须在 worker 的 io_context 上运行 |
| system_state | instance() | 是 | Meyers' Singleton |
| system_state | mark_started() | 是 | atomic exchange |
| system_state | snapshot() | 是 | atomic load + 时间计算 |

## 与旧版 stats::state 的迁移对照

| 旧版 (instance::worker::stats::state) | 新版 (stats::runtime::worker_load) | 变化 |
|--------------------------------------|-----------------------------------|------|
| `state::session_open()` | `worker_load::session_open()` | 命名空间变更 |
| `state::session_close()` | `worker_load::session_close()` | 命名空间变更 |
| `state::handoff_push()` | `worker_load::handoff_push()` | 命名空间变更 |
| `state::observe(ioc)` | `worker_load::observe(ioc)` | 算法零改动 |
| `state::snapshot()` | `worker_load::snapshot()` | 返回类型相同 |

迁移完全兼容，接口和算法与旧版一致。

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/snapshot|snapshot]] -- worker_snapshot 和 runtime_snapshot 定义
- [[core/stats/counter|counter]] -- 底层原子原语
- [[core/stats/gauge|gauge]] -- observe() 中使用的 EMA 平滑原理
- [[core/stats/traffic|traffic]] -- 互补的流量统计
- [[core/instance/worker/worker|Worker]] -- worker_load 的持有者
- [[core/instance/front/balancer|负载均衡器]] -- worker_snapshot 的消费者
