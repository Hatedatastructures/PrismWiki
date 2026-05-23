---
title: balancer — 负载均衡器
layer: core
source: include/prism/instance/front/balancer.hpp
created: 2026-05-17
tags: [agent, front, balancer, load-balancing, affinity]
---

# balancer — 负载均衡器

> 源码: `include/prism/instance/front/balancer.hpp`

## 概述

`balancer` 是前端层连接分发核心，基于加权评分的工作线程选择算法，支持过载检测与全局反压机制。综合活跃会话数、待处理移交数、事件循环延迟三维度进行动态调度。

### 核心职责

| 职责 | 说明 |
|------|------|
| 负载感知 | 实时采集各 worker 运行状态快照 |
| 智能选择 | 基于亲和性和负载评分选择最优 worker |
| 过载检测 | 滞后机制识别并标记过载 worker |
| 全局背压 | 系统整体过载时触发反压信号 |

### 调用关系

```
[[core/instance/front/listener|listener]]
    │
    ├─► select(affinity) ──► {worker_index, overflowed, backpressure}
    │       │
    │       ├─► mix_hash()      // 哈希混合
    │       ├─► score()         // 负载评分
    │       └─► refresh_state() // 过载状态更新
    │
    └─► dispatch(index, socket) ──► [[core/instance/worker/worker|worker]].dispatch_socket()
```

---

## 结构体定义

### worker_load_snapshot

```cpp
struct worker_load_snapshot
{
    std::uint32_t active_sessions{0};   // 当前活跃会话数
    std::uint32_t pending_handoffs{0};  // 等待处理的移交数
    std::uint64_t event_loop_lag_us{0}; // 事件循环延迟（微秒）
};
```

**数据来源**:

| 字段 | 来源 |
|------|------|
| `active_sessions` | `stats::state::session_counter()` |
| `pending_handoffs` | `stats::state::pending_handoffs_` |
| `event_loop_lag_us` | `stats::state::observe()` EMA 平滑结果 |

### distribute_config

```cpp
struct distribute_config
{
    double enter_overload{0.90};                // 进入过载阈值
    double exit_overload{0.80};                 // 退出过载阈值
    double global_backpressure_threshold{0.95}; // 全局反压阈值
    double weight_session{0.60};                // 会话数权重
    double weight_pending{0.10};                // 待处理数权重
    double weight_lag{0.30};                    // 延迟权重
    std::uint32_t session_capacity{1024};       // 会话容量基准
    std::uint32_t pending_capacity{256};        // 待处理容量基准
    std::uint64_t lag_capacity_us{5000};        // 延迟容量基准（5ms）
};
```

**参数关系**:

| 参数组 | 关系 |
|--------|------|
| 滞后机制 | `enter_overload > exit_overload` 防抖动 |
| 权重约束 | `weight_session + weight_pending + weight_lag ≈ 1.0` |
| 归一化 | 容量参数将绝对值转换为相对比例 |

---

## 类定义

```cpp
class balancer
{
public:
    struct worker_binding
    {
        std::function<void(tcp::socket)> dispatch;      // 分发函数
        std::function<worker_load_snapshot()> snapshot; // 快照函数
    };

    struct select_result
    {
        std::size_t worker_index{0}; // 选中的 worker 索引
        bool overflowed{false};      // 已过载标志
        bool backpressure{false};    // 全局反压标志
    };

    explicit balancer(memory::vector<worker_binding> bindings,
                      const distribute_config &config = {},
                      memory::resource_pointer mr = memory::current_resource());

    [[nodiscard]] auto select(std::uint64_t affinity_value) noexcept -> select_result;
    void dispatch(std::size_t worker_index, tcp::socket socket) const;
    [[nodiscard]] auto size() const noexcept -> std::size_t;

private:
    [[nodiscard]] static constexpr auto mix_hash(std::uint64_t value) noexcept -> std::uint64_t;
    [[nodiscard]] auto score(const worker_load_snapshot &snapshot) const noexcept -> double;
    void refresh_state(std::size_t worker_index, double load_score) noexcept;

    memory::vector<worker_binding> bindings_;     // 绑定列表
    memory::vector<std::uint8_t> overload_state_; // 过载状态（0=正常，1=过载）
    distribute_config config_;                    // 配置参数
    memory::resource_pointer mr_;                 // 内存资源
};
```

---

## 函数详解

### select() 函数

```cpp
[[nodiscard]] auto select(std::uint64_t affinity_value) noexcept -> select_result;
```

**职责**: 根据亲和性值选择最优 worker，返回选择结果。

**逐行解释**（复杂函数）:

```cpp
// ===== Step 1: 边界检查 =====
if (bindings_.empty()):
    return {}    // 返回默认值 {0, false, false}

const workers_count = bindings_.size()

// ===== Step 2: 双候选计算 =====

// Primary: 亲和性首选
const primary = mix_hash(affinity_value) % workers_count

// Secondary: 备选（确保不同）
std::size_t secondary = primary
if (workers_count > 1):
    secondary = mix_hash(affinity_value ^ 0xa24baed4963ee407ULL) % workers_count
    if (secondary == primary):
        secondary = (secondary + 1) % workers_count  // 强制不同

// ===== Step 3: 遍历评分 =====

double primary_score = 0.0
double secondary_score = 0.0
double min_score = MAX_DOUBLE
std::size_t overloaded_count = 0

for (index = 0; index < workers_count; ++index):
    
    // 获取负载快照
    snapshot = bindings_[index].snapshot()
    
    // 计算评分
    load_score = score(snapshot)
    
    // 更新过载状态（滞后机制）
    refresh_state(index, load_score)
    
    // 记录候选评分
    if (index == primary):
        primary_score = load_score
    if (index == secondary):
        secondary_score = load_score
    
    // 记录最低评分
    if (load_score < min_score):
        min_score = load_score
    
    // 统计过载数量
    if (overload_state_[index] != 0):
        ++overloaded_count

// ===== Step 4: 选择逻辑 =====

const primary_overloaded = (overload_state_[primary] != 0)
std::size_t selected = primary

if (primary_overloaded && workers_count > 1):
    // Primary 过载时，选择负载更低的 secondary
    selected = (secondary_score < primary_score) ? secondary : primary

// ===== Step 5: 背压判断 =====

const backpressure = (overloaded_count == workers_count) || 
                     (min_score >= config_.global_backpressure_threshold)

return {selected, primary_overloaded, backpressure}
```

**选择策略**:

| 场景 | 选择 |
|------|------|
| Primary 正常 | 选择 primary（保持亲和性） |
| Primary 过载，Secondary 更空闲 | 选择 secondary（负载迁移） |
| Primary 过载，Secondary 也过载 | 选择 primary（保持亲和性） |
| 所有 worker 过载 | 触发背压 + 选择负载最低者 |

---

### mix_hash() 函数

```cpp
[[nodiscard]] static constexpr auto mix_hash(std::uint64_t value) noexcept -> std::uint64_t;
```

**职责**: splitmix64 哈希混合，将亲和性值均匀打散。

**逐行解释**:

```cpp
// splitmix64 算法
value += 0x9e3779b97f4a7c15ULL                    // Step 1: 加黄金比例常数
value = (value ^ (value >> 30U)) * 0xbf58476d1ce4e5b9ULL  // Step 2: 右移30位 + XOR + 乘法
value = (value ^ (value >> 27U)) * 0x94d049bb133111ebULL  // Step 3: 右移27位 + XOR + 乘法
value = value ^ (value >> 31U)                    // Step 4: 右移31位 + XOR
return value
```

**设计目的**:
- 打散 IP 地址聚集效应（即使客户端集中在同一子网）
- `constexpr` 可编译期优化
- splitmix64 是公认的高质量哈希混合函数

---

### score() 函数

```cpp
[[nodiscard]] auto score(const worker_load_snapshot &snapshot) const noexcept -> double;
```

**职责**: 计算单个 worker 的综合负载评分。

**逐行解释**:

```cpp
// 防止除零（容量为 0 时回退到 1）
const session_capacity = max(1U, config_.session_capacity)
const pending_capacity = max(1U, config_.pending_capacity)
const lag_capacity = max(1ULL, config_.lag_capacity_us)

// 计算各维度比例
const session_ratio = snapshot.active_sessions / session_capacity
const pending_ratio = snapshot.pending_handoffs / pending_capacity
const lag_ratio = snapshot.event_loop_lag_us / lag_capacity

// 加权评分
return session_ratio * config_.weight_session +    // 60% 权重
       pending_ratio * config_.weight_pending +    // 10% 权重
       lag_ratio * config_.weight_lag              // 30% 权重
```

**评分公式**:

```
score = (active_sessions/1024) * 0.6 + (pending_handoffs/256) * 0.1 + (lag_us/5000) * 0.3
```

**评分范围**:

| 评分 | 含义 |
|------|------|
| 0.0 | 完全空闲 |
| 0.80 | 接近退出过载阈值 |
| 0.90 | 进入过载阈值 |
| 0.95 | 接近全局背压阈值 |
| > 1.0 | 超出容量基准 |

---

### refresh_state() 函数

```cpp
void refresh_state(std::size_t worker_index, double load_score) noexcept;
```

**职责**: 更新工作线程过载状态（滞后机制）。

**逐行解释**:

```cpp
// 获取当前状态
const current = (overload_state_[worker_index] != 0)

// 滞后机制：进入阈值高于退出阈值
if (!current && load_score >= config_.enter_overload):
    // 正常 → 过载：评分 >= 0.90
    overload_state_[worker_index] = 1
    return

if (current && load_score <= config_.exit_overload):
    // 过载 → 正常：评分 <= 0.80
    overload_state_[worker_index] = 0
```

**滞后机制示意**:

```
负载评分 ─────────────────────────────────────►
         0.80      0.90      0.95
         │         │         │
    ┌────┴─────────┴─────────┴────┐
    │ 正常区域 │ 过载区域 │ 背压区域│
    │ (状态=0) │ (状态=1) │ 全过载 │
    └──────────────────────────────┘

状态转换:
- 正常→过载: 评分 >= 0.90 (进入阈值)
- 过载→正常: 评分 <= 0.80 (退出阈值)
- 0.80~0.90 之间: 状态保持不变 (滞后区)
```

---

### dispatch() 函数

```cpp
void dispatch(std::size_t worker_index, tcp::socket socket) const;
```

**职责**: 将 socket 移交给目标 worker。

**逐行解释**:

```cpp
// 边界检查
if (bindings_.empty()):
    return    // 空绑定列表，直接返回

// 越界兜底
if (worker_index >= bindings_.size()):
    worker_index = 0    // 回退到 worker 0

// 调用分发函数
bindings_[worker_index].dispatch(std::move(socket))
// socket 所有权转移给 worker
```

**设计要点**:
- 越界索引安全回退到 worker 0
- `std::move` 转移 socket 所有权
- dispatch 函数将 socket 投递到目标 worker 的 io_context

---

## 调用链

### 上游调用者

| 调用者 | 调用时机 |
|--------|----------|
| [[core/instance/front/listener|listener]] | `accept_loop()` 中调用 `select()` |
| [[core/instance/front/listener|listener]] | 获取选择结果后调用 `dispatch()` |

### 下游依赖

| 依赖模块 | 调用方式 |
|----------|----------|
| [[core/instance/worker/worker|worker]] | `bindings_[i].dispatch(socket)` |
| [[core/instance/worker/worker|worker]] | `bindings_[i].snapshot()` |

### 完整调用链

```
listener.cpp:125  dispatcher_.select(affinity)
    │
    ├─► balancer::select()
    │       ├─► mix_hash(affinity)              // 双候选计算
    │       ├─► bindings_[i].snapshot()         // 获取快照
    │       │       └─► [[core/instance/worker/stats|stats]].load_snapshot()
    │       ├─► score(snapshot)                 // 评分计算
    │       └─► refresh_state(i, score)         // 过载状态更新
    │
    ▼
listener.cpp:135  dispatcher_.dispatch(index, socket)
    │
    └─► balancer::dispatch()
            └─► bindings_[i].dispatch(socket)
                    └─► [[core/instance/worker/worker|worker]].dispatch_socket()
```

---

## 设计决策

### 三维度评分权重

| 维度 | 权重 | 理由 |
|------|------|------|
| 会话数 | 60% | 主要处理压力指标 |
| 延迟 | 30% | 事件循环阻塞程度 |
| 待处理 | 10% | 队列积压情况 |

### 双候选策略

**选择**: primary + secondary 双候选，而非单候选。

**理由**:
- Primary: 亲和性首选，同一客户端尽量落到同一 worker
- Secondary: 备选，primary 过载时降级选择
- 避免单点热点，同时保持亲和性

### 滞后过载检测

**选择**: 进入阈值(0.90)高于退出阈值(0.80)。

**理由**:
- 防止负载在阈值附近波动时频繁切换状态
- 减少不必要的负载迁移开销
- 0.80~0.90 区间为滞后区，状态保持不变

---

## 配置调优

### 场景建议

| 场景 | 建议配置 |
|------|----------|
| 高并发短连接 | 降低 weight_lag，提高 weight_session |
| 长连接会话 | 提高 session_capacity，降低阈值 |
| 低延迟优先 | 提高 weight_lag，降低 lag_capacity_us |
| 高吞吐优先 | 提高 weight_session，提高 session_capacity |

### Worker 数量规划

| CPU 核心数 | 建议 Worker 数量 |
|------------|------------------|
| 2 核 | 1（保留监听线程） |
| 4 核 | 3 |
| 8 核 | 7 |
| 16+ 核 | 核心 - 1 |

---

## 相关文档

- [[core/instance/front/listener|listener]] — 监听器
- [[core/instance/worker/worker|worker]] — 工作线程
- [[core/instance/worker/stats|stats]] — 负载统计
- [[startup]] — 启动流程