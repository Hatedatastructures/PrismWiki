---
title: Stats 统计模块总览
layer: core
source:
  - include/prism/stats/counter.hpp
  - include/prism/stats/gauge.hpp
  - include/prism/stats/traffic.hpp
  - include/prism/stats/runtime.hpp
  - include/prism/stats/snapshot.hpp
  - include/prism/stats/memory.hpp
  - include/prism/stats/account.hpp
  - src/prism/stats/runtime.cpp
module: stats
tags: [stats, overview, atomic, lock-free, cache-line]
updated: 2026-05-29
---

# Stats 统计模块总览

> 源码: `include/prism/stats/` | 实现: `src/prism/stats/`

Prism 的运行时统计基础设施，零分配、无锁、缓存行对齐。被 [[core/instance/worker/worker|Worker]]、[[core/instance/front/balancer|负载均衡器]]、[[core/connect/tunnel/tunnel|Tunnel]]、[[core/multiplex/core|Mux Core]] 广泛使用。

## 设计原则

- **零分配**: 热路径统计操作使用 `fetch_add`/`fetch_sub`，无动态内存分配
- **无锁并发**: 全部 `std::atomic` + `memory_order_relaxed`
- **缓存行对齐**: `alignas(64)` 独占缓存行，防止 false sharing
- **松散一致**: 快照是瞬时值的近似拷贝，不保证跨字段原子性
- **单写者设计**: per-worker 单写者，减少跨核原子操作开销

## 模块组成

| 模块 | 源码 | 说明 |
|------|------|------|
| [[core/stats/counter\|counter]] | `counter.hpp` | 缓存行对齐原子计数器（`fetch_add`/`fetch_sub`） |
| [[core/stats/gauge\|gauge]] | `gauge.hpp` | EMA 平滑瞬时值（alpha=7/8，非线程安全） |
| [[core/stats/snapshot\|snapshot]] | `snapshot.hpp` | 所有快照类型定义（worker/runtime/traffic/protocol/memory） |
| [[core/stats/runtime\|runtime]] | `runtime.hpp` | worker_load（负载监控 + EMA 延迟观测）+ system_state（全局运行状态） |
| [[core/stats/traffic\|traffic]] | `traffic.hpp` | per-worker 流量计数器 + 全局 COW 聚合 |
| [[core/stats/memory\|memory]] | `memory.hpp` | 全局 PMR 池分配统计追踪器 |
| [[core/stats/account\|account]] | `account.hpp` | 账户统计观察者 |

## 设计决策（WHY）

### 为什么快照不保证跨字段原子性？

**问题**: `traffic_snapshot` 包含 `total_connections`、`total_active`、`total_uplink` 等多个字段，消费者可能需要一致的全局视图。

**选择**: 使用 `memory_order_relaxed` 逐字段读取，不提供跨字段原子性保证。跨字段原子性需要全局锁或 seqlock，代价是写入路径的开销。Stats 的消费者（监控面板、日志、balancer 调度）只需要近似值——例如 `total_connections` 比 `total_active` 多 1-2 是可接受的。

**后果**: 读取快照时 `total_active` 可能比 `total_connections` 大（连接建立但 active 还未递增）。消费者需容忍这种不一致。

### 为什么 traffic_state 用批量刷入？

**问题**: 数据转发热路径中每字节的流量统计如果逐字节 `fetch_add`，原子操作次数与传输量成正比（O(bytes)），严重影响吞吐。

**选择**: 热路径中流量在局部变量累积，会话/子流结束时一次 `flush_traffic()` 最多 4 次 `fetch_add`（total_uplink, total_downlink, proto_uplink, proto_downlink）。单次会话总计 10 次原子操作，相比逐字节的数千次，开销从 O(bytes) 降到 O(1)。

**后果**: 会话中途崩溃时，未刷入的流量数据丢失。这对于监控场景可接受（崩溃本身是异常事件）。

### 为什么 gauge 非线程安全？

**问题**: gauge 用于延迟测量和负载评估，需要高效更新。

**选择**: gauge 使用普通 `double` 成员，无 `std::atomic`。它仅在 worker 内部的 `observe()` 协程中使用（单写者场景），避免了 `std::atomic<double>` 的性能开销和可移植性问题。

**后果**: gauge 只能在单一线程中使用。跨线程读取 gauge 值是数据竞争。如果未来需要跨线程访问，需要改为 `std::atomic<double>` 或独立 gauge 实例 + 聚合。

### 为什么 COW 注册表不释放旧 vector？

**问题**: `traffic_state::aggregate()` 需要遍历所有 worker 的计数器，但 worker 数量在运行时固定。

**选择**: `register_instance()` 使用 Copy-on-Write 模式——创建新的 vector 替换旧的，旧的 vector 不释放。原因是有并发的 `aggregate()` 可能正在读旧 vector，释放会导致 use-after-free。

**后果**: 每次注册产生一个 leaked vector（通常 < 64 个 worker，每个 vector 几十字节），内存泄漏可忽略。Worker 生命周期等于进程生命周期，不会反复注册。

## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| counter 必须 alignas(64) | 每个实例独占 64 字节缓存行 | false sharing 导致性能退化（跨核 fetch_add 从 1ns 升至 6ns+） | `counter.hpp` |
| gauge 仅限单线程 | 无 atomic 保护 | 多线程同时 update/read 是数据竞争（UB） | `gauge.hpp` |
| traffic_state 单写者 | 设计上仅由所属 worker 写入 | 多写者不会崩溃（atomic 本身线程安全），但批量刷入语义失效 | `traffic.hpp` |
| snapshot 不保证一致性 | 逐字段 relaxed load | 跨字段计算可能产生逻辑不一致 | `snapshot.hpp` |
| COW 注册表不回收 | 旧 vector 不释放 | 内存微量泄漏，worker 数量有限可忽略 | `traffic.hpp` |
| protocol_slot 2 个占一行 | `alignas(64)` 使 2 个 slot 共享一行 | 超过 2 个协议维度会跨缓存行，性能退化 | `traffic.hpp:30` |
| slot_count=16 | 预留 16 个协议槽位 | `protocol_type` 枚举值超过 16 时越界访问 | `snapshot.hpp:43` |

## 故障场景

### 1. Worker 崩溃导致该 worker 统计丢失

**触发条件**: Worker 线程崩溃或被 kill

**传播路径**: 该 worker 的 `traffic_state` 析构 → 局部累积的流量数据丢失 → `aggregate()` 遍历时不包含该 worker

**外部表现**: 全局流量统计偏低，`total_active` 比实际少

**恢复机制**: 其他 worker 不受影响。`aggregate()` 遍历的 COW 注册表持有裸指针，worker 析构后该指针悬挂——但 worker 生命周期等于进程生命周期，正常情况不会发生

### 2. EMA 平滑导致突发负载不敏感

**触发条件**: 突发大量连接涌入某个 worker

**传播路径**: `observe()` 每 250ms 采样一次 → EMA alpha=7/8 平滑 → `lag_us_` 变化缓慢

**外部表现**: balancer 读取的 `lag_us` 可能低估当前负载，继续向该 worker 分发

**恢复机制**: 随时间推移 EMA 会收敛到真实值。前 16 次采样为预热期

### 3. flush_traffic 数据丢失

**触发条件**: 会话中途异常退出（未执行 flush_traffic）

**传播路径**: tunnel/mux 结束时调用 `flush_traffic()` → 异常路径跳过 flush → 累积流量丢失

**外部表现**: 流量统计偏低，`uplink_bytes`/`downlink_bytes` 不完整

### 4. memory_tracker current_usage 下溢

**触发条件**: instrumented memory resource 的 deallocate 调用与实际分配不匹配

**传播路径**: `current_usage_.fetch_sub(bytes)` → 如果 bytes 大于实际分配量 → 值下溢为大正数

**外部表现**: `current_usage` 显示异常大的值

**恢复机制**: 重启进程重置计数器

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| [[core/instance/worker/worker\|worker]] ← stats | 被依赖 | worker 持有 `worker_load` 和 `traffic_state`，构造时注册到 COW 注册表 |
| [[core/instance/worker/launch\|launch]] ← stats | 被依赖 | launch 调用 `on_connect()`、`session_open()`、`on_protocol_detected()` |
| [[core/instance/front/balancer\|balancer]] ← stats | 被依赖 | balancer 读取 `worker_snapshot`（active_sessions、lag_us）做调度决策 |
| [[core/connect/tunnel/tunnel\|tunnel]] ← stats | 被依赖 | tunnel 结束时调用 `flush_traffic()` 批量刷入流量数据 |
| [[core/multiplex/core\|mux core]] ← stats | 被依赖 | mux 子流结束时调用 `flush_traffic()` |
| stats ← [[core/memory/overview\|memory]] | 依赖 | `memory_tracker` 包装全局 PMR 池，在分配/释放时更新计数器 |
| stats ← [[core/protocol/types\|protocol]] | 依赖 | `protocol_type` 枚举作为 `protocol_slot` 数组索引 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| `counter`/`gauge` 接口变更 | 所有热路径模块 | worker、tunnel、mux、launch 编译失败 |
| `snapshot` 结构体字段增删 | 所有消费者 | balancer 调度逻辑、监控面板需适配 |
| `slot_count` 变更 | `protocol_slot` 数组 | 增大浪费内存，减小导致越界 |
| `flush_traffic()` 参数变更 | tunnel/mux 调用方 | 编译失败或统计维度丢失 |
| COW 注册表机制变更 | `aggregate()` 行为 | 监控数据一致性受影响 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| `protocol_type` 枚举新增值 | `slot_count` 是否足够 | `snapshot.hpp:43` 的 slot_count 常量 |
| worker 线程数变更 | COW 注册表大小 | 内存泄漏量（每次注册一个 vector） |
| PMR 池替换为其他分配器 | `memory_tracker` 的 instrumented MR | 是否还能正确拦截分配/释放 |
| balancer 调度算法变更 | `worker_snapshot` 字段充分性 | 是否需要新增延迟/队列指标 |

## 线程安全模型

| 组件 | 写入者 | 读取者 | 机制 |
|------|--------|--------|------|
| `counter` | 多写者 | 多读者 | `atomic` + relaxed |
| `gauge` | 单写者 | 单读者 | 非线程安全，worker 内部 |
| `worker_load` | 单写者(worker) | 多读者(balancer) | `atomic` + relaxed |
| `traffic_state` | 单写者(worker) | 多读者(aggregate) | `atomic` + relaxed |
| COW 注册表 | worker 构造时 | aggregate() | COW + acquire/release |
| `memory_tracker` | 多写者(PMR) | 多读者 | `atomic` + relaxed |
| `system_state` | main() 初始化 | 多读者 | `atomic` + relaxed |

## 相关文档

- [[core/instance/worker/worker|Worker]] — 持有 worker_load 和 traffic_state
- [[core/instance/worker/launch|Launch]] — 调用 on_connect/session_open
- [[core/instance/front/balancer|Balancer]] — 使用 worker_snapshot 做调度
- [[core/connect/tunnel/tunnel|Tunnel]] — 调用 flush_traffic
- [[core/multiplex/core|Mux Core]] — 调用 flush_traffic
- [[core/memory/overview|Memory]] — PMR 池分配统计
