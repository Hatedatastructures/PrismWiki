---
title: Stats 统计模块总览
layer: core
source:
  - I:/code/Prism/include/prism/stats/stats.hpp
module: stats
tags: [stats, overview]
updated: 2026-05-27
---

# Stats 统计模块总览

Prism 的运行时统计基础设施，零分配、无锁、缓存行对齐。被 [[core/instance/worker/worker|Worker]]、[[core/instance/front/balancer|负载均衡器]]、[[core/pipeline/overview|协议处理器]] 广泛使用。

## 设计原则

- **零分配**: 热路径统计操作使用 `fetch_add`/`fetch_sub`，无动态内存分配
- **无锁并发**: 全部 `std::atomic` + `memory_order_relaxed`
- **缓存行对齐**: `alignas(64)` 独占缓存行，防止 false sharing
- **松散一致**: 快照是瞬时值的近似拷贝，不保证跨字段原子性
- **单写者设计**: per-worker 单写者，减少跨核原子操作开销

## 模块组成

| 模块 | 源码 | 说明 |
|------|------|------|
| [[core/stats/counter\|counter]] | `counter.hpp` | 缓存行对齐原子计数器 |
| [[core/stats/gauge\|gauge]] | `gauge.hpp` | EMA 平滑瞬时值（alpha=7/8） |
| [[core/stats/snapshot\|snapshot]] | `snapshot.hpp` | 所有快照类型定义 |
| [[core/stats/runtime\|runtime]] | `runtime.hpp` | Worker 负载 + 全局运行状态 |
| [[core/stats/traffic\|traffic]] | `traffic.hpp` | Per-worker 流量 + 全局 COW 聚合 |
| [[core/stats/account\|account]] | `account.hpp` | 账户统计观察者 |

## 设计决策

### 为什么快照不保证跨字段原子性？

跨字段原子性需要全局锁或 seqlock，代价是写入路径的开销。Stats 的消费者（监控面板、日志、balancer 调度）只需要近似值。例如 `total_connections` 比 `total_active` 多 1-2 是可接受的。

**后果**: 读取快照时 `total_active` 可能比 `total_connections` 大（连接建立但 active 还未递增）。消费者需容忍。

### 为什么 traffic_state 用批量刷入？

热路径中流量在局部变量累积，会话结束时一次 `flush_traffic()` 最多 4 次 `fetch_add`。单次会话总计 10 次原子操作，相比逐字节刷入的数千次，开销从 O(bytes) 降到 O(1)。

**后果**: 会话中途崩溃时，未刷入的流量数据丢失。

## 线程安全模型

| 组件 | 写入者 | 读取者 | 机制 |
|------|--------|--------|------|
| `counter` | 多写者 | 多读者 | `atomic` + relaxed |
| `gauge` | 单写者 | 单读者 | 非线程安全，worker 内部 |
| `worker_load` | 单写者(worker) | 多读者(balancer) | `atomic` + relaxed |
| `traffic_state` | 单写者(worker) | 多读者(aggregate) | `atomic` + relaxed |
| COW 注册表 | worker 构造时 | aggregate() | COW + acquire/release |

## 引用关系

### 被引用

- [[core/instance/worker/worker|worker]]：持有 `worker_load` 和 `traffic_state`
- [[core/instance/worker/launch|launch]]：调用 `on_connect()`, `session_open()`
- [[core/instance/front/balancer|balancer]]：使用 `worker_load_snapshot` 做调度决策
- [[core/connect/tunnel/tunnel|tunnel]]：调用 `flush_traffic()`
- [[core/multiplex/core|mux core]]：调用 `flush_traffic()`
