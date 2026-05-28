---
title: balancer -- 负载均衡器
layer: core
source: include/prism/instance/front/balancer.hpp
created: 2026-05-17
updated: 2026-05-28
tags: [front, balancer, load-balancing, affinity]
---

# balancer -- 负载均衡器

> 源码: `include/prism/instance/front/balancer.hpp`

## 概述

前端层连接分发核心，基于加权评分的工作线程选择算法，支持过载检测与全局反压机制。综合活跃会话数、待处理移交数、事件循环延迟三维度进行动态调度。

| 职责 | 说明 |
|------|------|
| 负载感知 | 实时采集各 worker 运行状态快照 |
| 智能选择 | 基于亲和性和负载评分选择最优 worker |
| 过载检测 | 滞后机制识别并标记过载 worker |
| 全局背压 | 系统整体过载时触发反压信号 |

### 调用关系

- 调用者: [[core/instance/front/listener|listener]]
- 被调用者: [[core/instance/worker/worker|worker]] (通过 `worker_binding.dispatch` 和 `.snapshot`)

---

## 配置参数

### distribute_config

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enter_overload` | 0.90 | 进入过载阈值 |
| `exit_overload` | 0.80 | 退出过载阈值 |
| `backpressure_thresh` | 0.95 | 全局反压阈值 |
| `weight_session` | 0.60 | 会话数权重 |
| `weight_pending` | 0.10 | 待处理数权重 |
| `weight_lag` | 0.30 | 延迟权重 |
| `session_capacity` | 1024 | 会话容量基准 |
| `pending_capacity` | 256 | 待处理容量基准 |
| `lag_cap` | 5000 | 延迟容量基准 (us) |

**参数约束**:

- type: 不变量 — `enter_overload > exit_overload`（滞后防抖动）
- type: 归一化 — `weight_session + weight_pending + weight_lag` 应接近 1.0
- type: 归一化 — 容量参数将绝对值转换为相对比例用于评分

---

## 核心接口

### balancer 类

| 方法 | 签名 | 说明 |
|------|------|------|
| 构造 | `(vector<worker_binding>, distribute_config, mr)` | 初始化绑定和过载状态 |
| `select` | `(uint64_t affinity) noexcept -> select_result` | 根据亲和性选择最优 worker |
| `dispatch` | `(size_t index, tcp::socket) const` | 将 socket 移交给目标 worker |
| `size` | `() noexcept -> size_t` | worker 数量 |

### worker_binding

| 成员 | 类型 | 说明 |
|------|------|------|
| `dispatch` | `function<void(tcp::socket)>` | 分发函数（将 socket 投递到 worker io_context） |
| `snapshot` | `function<stats::worker_snapshot()>` | 负载快照获取函数 |

### select_result

| 成员 | 类型 | 说明 |
|------|------|------|
| `worker_index` | `size_t` | 选中的 worker 索引 |
| `overflowed` | `bool` | 选中 worker 已过载标志 |
| `backpressure` | `bool` | 全局反压标志 |

---

## 选择算法

### select() 流程

1. 计算双候选：primary = `mix_hash(affinity) % N`，secondary 使用不同种子确保与 primary 不同
2. 遍历所有 worker：获取快照、计算评分、更新过载状态
3. 若 primary 过载且 secondary 评分更低，选 secondary
4. 若所有 worker 过载或最低评分 >= `backpressure_thresh`，触发背压

### 选择策略

| 场景 | 选择 |
|------|------|
| Primary 正常 | 选择 primary（保持亲和性） |
| Primary 过载，Secondary 更空闲 | 选择 secondary（负载迁移） |
| Primary 过载，Secondary 也过载 | 选择 primary（保持亲和性） |
| 所有 worker 过载 | 触发背压 + 选择负载最低者 |

### 为什么用双候选而非单候选

**理由**: 亲和性保证同一客户端落到同一 worker，但 primary 过载时需要降级选择。secondary 通过不同哈希种子计算，避免热点聚集。

**后果**: 正常情况下亲和性稳定；过载时自动迁移到较空闲的 worker，牺牲亲和性换取负载均衡。

---

## 评分公式

```
score = (active_sessions / session_capacity) * weight_session
      + (pending_handoffs / pending_capacity) * weight_pending
      + (event_loop_lag_us / lag_cap) * weight_lag
```

默认: `(sessions/1024)*0.6 + (pending/256)*0.1 + (lag_us/5000)*0.3`

| 评分 | 含义 |
|------|------|
| 0.0 | 完全空闲 |
| 0.80 | 接近退出过载阈值 |
| 0.90 | 进入过载阈值 |
| 0.95 | 接近全局背压阈值 |

### 为什么会话数权重 60%

**理由**: 会话数是最直接的处理压力指标。延迟占 30%（事件循环阻塞程度），待处理数仅 10%（瞬时队列积压，通常波动大但影响较小）。

---

## 过载检测

采用滞后机制：进入阈值(0.90)高于退出阈值(0.80)。0.80~0.90 区间为滞后区，状态保持不变，防止阈值附近频繁切换。

| 转换 | 条件 |
|------|------|
| 正常 -> 过载 | score >= `enter_overload` (0.90) |
| 过载 -> 正常 | score <= `exit_overload` (0.80) |

### 为什么用滞后机制

**理由**: 防止负载在阈值附近波动时频繁切换状态，减少不必要的负载迁移开销。

**约束**:
- type: 不变量 — `enter_overload` 必须严格大于 `exit_overload`
- violation: 两者相等或倒置将导致过载检测失效

---

## dispatch() 行为

越界索引安全回退到 worker 0。`std::move` 转移 socket 所有权，dispatch 函数将 socket 投递到目标 worker 的 io_context。

---

## 线程安全

- type: 约束 — balancer 不是线程安全的，调用方需确保在同一线程上下文中操作（listener 线程）
- `dispatch` 函数可能抛出异常，调用方需妥善处理

## 配置调优建议

| 场景 | 建议 |
|------|------|
| 高并发短连接 | 降低 weight_lag，提高 weight_session |
| 长连接会话 | 提高 session_capacity，降低阈值 |
| 低延迟优先 | 提高 weight_lag，降低 lag_cap |
| 高吞吐优先 | 提高 weight_session，提高 session_capacity |

## 相关文档

- [[core/instance/front/listener|listener]] -- 监听器
- [[core/instance/worker/worker|worker]] -- 工作线程
- [[core/instance/worker/stats|stats]] -- 负载统计
- [[startup]] -- 启动流程
