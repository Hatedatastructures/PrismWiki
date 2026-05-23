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

## 类定义

### worker_load

```cpp
namespace psm::stats::runtime
{
    namespace net = boost::asio;

    class worker_load
    {
    public:
        worker_load();

        void session_open() noexcept;
        void session_close() noexcept;
        void handoff_push() noexcept;
        void handoff_pop() noexcept;

        [[nodiscard]] auto session_counter() const noexcept
            -> const std::shared_ptr<std::atomic<std::uint32_t>> &;

        [[nodiscard]] auto snapshot() const noexcept -> worker_load_snapshot;

        auto observe(net::io_context &ioc) -> net::awaitable<void>;

    private:
        std::shared_ptr<std::atomic<std::uint32_t>> active_sessions_;
        std::atomic<std::uint32_t> pending_handoffs_{0};
        std::atomic<std::uint64_t> event_loop_lag_us_{0};
    };
}
```

### system_state

```cpp
class system_state
{
public:
    static auto instance() -> system_state &;
    void mark_started(std::uint32_t worker_count) noexcept;
    [[nodiscard]] auto snapshot() const noexcept -> runtime_snapshot;

private:
    std::atomic<bool> started_{false};
    std::chrono::steady_clock::time_point start_time_{};
    std::uint32_t worker_count_{0};
};
```

## worker_load 详解

### 活跃会话计数器 (active_sessions_)

```cpp
std::shared_ptr<std::atomic<std::uint32_t>> active_sessions_;
```

**为什么使用 shared_ptr 包装？**

会话关闭回调（`on_closed`）可能在 session 对象析构后才执行。如果回调直接捕获 `worker_load*` 裸指针，存在悬挂指针风险。通过 `shared_ptr<atomic<uint32_t>>` 包装计数器：

```cpp
// launch 中设置关闭回调
auto counter = metrics_.session_counter();  // shared_ptr 拷贝
auto on_closed = [counter]() {
    counter->fetch_sub(1, std::memory_order_relaxed);  // 安全，计数器独立于 worker 生命周期
};
```

即使 worker 对象已部分析构，回调持有的 `shared_ptr` 仍保证计数器有效。

### session_open / session_close

```cpp
void worker_load::session_open() noexcept
{
    active_sessions_->fetch_add(1U, std::memory_order_relaxed);
}

void worker_load::session_close() noexcept
{
    active_sessions_->fetch_sub(1U, std::memory_order_relaxed);
}
```

**调用（向上）**:
- `session_open()` 被 [[core/instance/worker/launch|launch::start()]] 调用
- `session_close()` 被 [[core/instance/worker/launch|launch::dispatch()]] 的 `on_closed` 回调调用

### handoff_push / handoff_pop

```cpp
void worker_load::handoff_push() noexcept
{
    pending_handoffs_.fetch_add(1U, std::memory_order_relaxed);
}

void worker_load::handoff_pop() noexcept
{
    pending_handoffs_.fetch_sub(1U, std::memory_order_relaxed);
}
```

**调用时序**:

```
listener.accept() ──► balancer.select()
                          │
                          ▼
                    dispatch_socket()
                     handoff_push()     ← pending = 1
                          │
                          ▼
                     ioc.post(task)
                          │
                          ▼ (worker 线程取出任务)
                     handoff_pop()      ← pending = 0
                          │
                          ▼
                     session_open()     ← active = 1
```

### observe() 协程

`observe()` 是事件循环延迟监测的核心协程，在 worker 的 `io_context` 上持续运行。

#### 算法流程

```
observe 协程启动
    │
    ▼
┌─── 预热阶段（16 次采样）───────────────────────────┐
│                                                      │
│  1. 设置 timer 到 expected_time + 250ms              │
│  2. co_await timer.async_wait()                      │
│  3. 测量 actual_time - expected_time 的偏差          │
│  4. 截断到 20ms 上限                                 │
│  5. 增量更新 jitter_baseline_us（移动平均）           │
│  6. event_loop_lag_us_ = 0（预热期间报告零延迟）     │
│                                                      │
└──────────────────────────────────────────────────────┘
    │
    ▼
┌─── 正常采样阶段（无限循环）─────────────────────────┐
│                                                      │
│  1. 设置 timer 到 expected_time + 250ms              │
│  2. co_await timer.async_wait()                      │
│  3. 测量 raw_lag = actual_time - expected_time       │
│  4. 截断: capped_lag = min(raw_lag, 20ms)            │
│  5. 扣除基线: effective = capped - jitter_baseline   │
│  6. 过滤小抖动: if effective < 1ms → effective = 0   │
│  7. EMA 平滑: smoothed = (smoothed * 7 + effective) / 8 │
│  8. 存储: event_loop_lag_us_ = smoothed              │
│                                                      │
└──────────────────────────────────────────────────────┘
```

#### 关键参数

| 参数 | 值 | 作用 |
|------|-----|------|
| 采样间隔 | 250ms | 平衡精度和开销 |
| 预热样本数 | 16 | 估算系统调度抖动基线 |
| EMA 系数 | 7/8 | 新样本权重 12.5%，历史权重 87.5% |
| 小抖动阈值 | 1ms (1000us) | 低于此值视为零延迟 |
| 上限截断 | 20ms (20000us) | 防止单次异常值污染 |

#### 预热阶段抖动基线

```cpp
// 预热阶段：增量平均
jitter_baseline_us = (jitter_baseline_us * warmup_samples + capped_lag_us) / (warmup_samples + 1U);
```

预热阶段计算的是"空负载下的期望调度延迟"，即操作系统定时器精度和 Asio 内部队列处理时间的基线。正常采样时，从这个基线中扣除，只保留"真实"的事件循环积压延迟。

#### 定时器错误处理

```cpp
boost::system::error_code ec;
co_await timer.async_wait(net::redirect_error(net::use_awaitable, ec));
if (ec)
{
    if (ec != net::error::operation_aborted)
        trace::debug("[Stats] observe timer error: {}", ec.message());
    co_return;
}
```

当 `io_context` 停止时，所有未完成的异步操作被取消，`ec` 为 `operation_aborted`。这是正常的退出路径。其他错误码记录 debug 日志后退出。

### snapshot()

```cpp
auto worker_load::snapshot() const noexcept -> worker_load_snapshot
{
    return {
        active_sessions_->load(std::memory_order_relaxed),
        pending_handoffs_.load(std::memory_order_relaxed),
        event_loop_lag_us_.load(std::memory_order_relaxed)
    };
}
```

**被调用（向上）**: [[core/instance/worker/worker|worker::load_snapshot()]]，进而被 [[core/instance/front/balancer|balancer]] 的 `score()` 方法消费。

## system_state 详解

### 单例模式

```cpp
auto system_state::instance() -> system_state &
{
    static system_state inst;
    return inst;
}
```

使用 Meyers' Singleton（C++11 保证线程安全的局部静态变量初始化）。首次调用 `instance()` 时构造，进程生命周期内持续存在。

### mark_started()

```cpp
void system_state::mark_started(std::uint32_t worker_count) noexcept
{
    if (started_.exchange(true, std::memory_order_relaxed))
        return;  // 已启动，重复调用为空操作
    start_time_ = std::chrono::steady_clock::now();
    worker_count_ = worker_count;
}
```

**防重复调用**: 使用 `exchange` 原子操作确保只初始化一次。后续调用（如配置热重载）不会重置启动时间。

**调用（向上）**: 在 `main.cpp` 启动流程中调用一次。

### snapshot()

```cpp
auto system_state::snapshot() const noexcept -> runtime_snapshot
{
    if (!started_.load(std::memory_order_relaxed))
        return {};  // 未启动时返回空快照
    const auto now = std::chrono::steady_clock::now();
    const auto uptime = std::chrono::duration_cast<std::chrono::seconds>(now - start_time_).count();
    return {static_cast<std::uint64_t>(uptime), worker_count_};
}
```

运行时间使用 `steady_clock` 计算，不受 NTP 调整或手动修改系统时间影响。

## 线程安全

| 组件 | 方法 | 线程安全 | 说明 |
|------|------|----------|------|
| worker_load | session_open() | 是 | `atomic fetch_add` |
| worker_load | session_close() | 是 | `atomic fetch_sub` |
| worker_load | handoff_push() | 是 | `atomic fetch_add` |
| worker_load | handoff_pop() | 是 | `atomic fetch_sub` |
| worker_load | snapshot() | 是 | `atomic load` x3 |
| worker_load | observe() | 否 | 必须在 worker 的 io_context 上运行 |
| system_state | instance() | 是 | Meyers' Singleton |
| system_state | mark_started() | 是 | `atomic exchange` |
| system_state | snapshot() | 是 | `atomic load` + 时间计算 |

## 与旧版 stats::state 的迁移对照

| 旧版 (instance::worker::stats::state) | 新版 (stats::runtime::worker_load) | 变化 |
|--------------------------------------|-----------------------------------|------|
| `state::session_open()` | `worker_load::session_open()` | 命名空间变更 |
| `state::session_close()` | `worker_load::session_close()` | 命名空间变更 |
| `state::handoff_push()` | `worker_load::handoff_push()` | 命名空间变更 |
| `state::observe(ioc)` | `worker_load::observe(ioc)` | 算法零改动 |
| `state::snapshot()` | `worker_load::snapshot()` | 返回类型相同 |

迁移完全兼容，接口和算法与旧版一致。参见 [[core/instance/worker/stats|旧版 Worker 负载统计]]。

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/snapshot|snapshot]] -- worker_load_snapshot 和 runtime_snapshot 定义
- [[core/stats/counter|counter]] -- 底层原子原语
- [[core/stats/gauge|gauge]] -- observe() 中使用的 EMA 平滑原理
- [[core/stats/traffic|traffic]] -- 互补的流量统计
- [[core/instance/worker/worker|Worker]] -- worker_load 的持有者
- [[core/instance/front/balancer|负载均衡器]] -- worker_load_snapshot 的消费者
