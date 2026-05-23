---
title: Counter 缓存行对齐原子计数器
layer: core
source: I:/code/Prism/include/prism/stats/counter.hpp
module: stats
tags: [stats, counter, atomic, performance]
created: 2026-05-23
updated: 2026-05-23
---

# Counter 缓存行对齐原子计数器

`counter` 是 Stats 模块最底层的原语，提供独占缓存行的 64 位无锁原子计数器。设计目标是在高并发递增场景下避免 false sharing，将跨核原子操作开销控制在 1ns~6ns。

> 源码: `include/prism/stats/counter.hpp`

## 类定义

```cpp
namespace psm::stats
{
    class alignas(64) counter
    {
    public:
        counter() = default;

        void increment(std::uint64_t n = 1) noexcept;
        void decrement(std::uint64_t n = 1) noexcept;
        [[nodiscard]] auto load() const noexcept -> std::uint64_t;
        [[nodiscard]] auto exchange(std::uint64_t desired) noexcept -> std::uint64_t;

    private:
        std::atomic<std::uint64_t> value_{0};
    };
}
```

## 设计决策

### 缓存行对齐 (alignas(64))

x86-64 架构的缓存行为 64 字节。当两个频繁修改的原子变量共享同一缓存行时，多核 CPU 需要在核间反复传递缓存行所有权（缓存一致性协议 MESI），导致严重的性能退化——这就是 false sharing。

```
未对齐（false sharing）:
┌──────────────── 64B cache line ────────────────┐
│ counter_A.value_ │ counter_B.value_ │ padding   │
└─────────────────────────────────────────────────┘
  Core 0 修改 A    Core 1 修改 B
  → 每次 modify 都使对方的 cache line 失效
  → 跨核延迟 ~40ns（vs 同核 ~1ns）

对齐后（无 false sharing）:
┌──── 64B cache line 1 ────┐┌──── 64B cache line 2 ────┐
│ counter_A.value_ + pad   ││ counter_B.value_ + pad   │
└──────────────────────────┘└──────────────────────────┘
  Core 0 修改 A             Core 1 修改 B
  → 独立缓存行，互不影响
```

`counter` 实际占用 64 字节（8 字节 atomic + 56 字节 padding），但每个实例独占缓存行，多核并发递增时不产生缓存行争用。

### memory_order_relaxed

所有操作使用 `memory_order_relaxed`，这是有意为之的设计决策：

| 内存序 | 开销 | 适用场景 |
|--------|------|----------|
| `relaxed` | 最低（无内存屏障） | 计数器递增/递减，仅关心最终值 |
| `acquire/release` | 中等（半屏障） | 生产者-消费者标志位 |
| `seq_cst` | 最高（全屏障） | 需要全局一致顺序的场景 |

统计计数器不需要与其他变量建立 happens-before 关系：
- `increment()` 仅需要原子递增，不保护任何关联数据
- `load()` 仅读取近似值，不需要看到最新的其他写入
- [[core/instance/front/balancer|负载均衡器]] 只需要近似值做调度决策

### counter 大小分析

```
┌────────────────────────────────────────────────────────────┐
│                    counter 内存布局                         │
│                                                            │
│  offset 0-7:   std::atomic<std::uint64_t> value_  (8 字节) │
│  offset 8-63:  padding                          (56 字节)  │
│                                                            │
│  总计: 64 字节 (alignas(64) 强制)                          │
└────────────────────────────────────────────────────────────┘
```

## 方法详解

### increment

```cpp
void increment(std::uint64_t n = 1) noexcept
{
    value_.fetch_add(n, std::memory_order_relaxed);
}
```

**调用（向下）**: `std::atomic<std::uint64_t>::fetch_add()`

**行为**: 原子地将 `value_` 增加 `n`。编译为单条 `lock xadd` 指令（x86-64）。

**性能**:
- 同核（单线程）: ~1ns
- 跨核（多线程）: ~6ns
- 不产生内存屏障

**被调用（向上）**: [[core/stats/traffic|traffic_state]] 的 `on_connect()`、`on_protocol_detected()`、`flush_traffic()`、`on_auth_success()` 等方法内部使用类似模式。

### decrement

```cpp
void decrement(std::uint64_t n = 1) noexcept
{
    value_.fetch_sub(n, std::memory_order_relaxed);
}
```

**行为**: 原子地将 `value_` 减少 `n`。编译为单条 `lock xadd` 指令（传入负数）。

**被调用（向上）**: [[core/stats/traffic|traffic_state]] 的 `on_disconnect()` 方法。

### load

```cpp
[[nodiscard]] auto load() const noexcept -> std::uint64_t
{
    return value_.load(std::memory_order_relaxed);
}
```

**行为**: 读取当前计数值。在 x86-64 上编译为普通 `mov` 指令（对齐的 64 位读取天然原子）。

**被调用（向上）**: [[core/stats/snapshot|snapshot]] 的构造过程中使用类似操作。

### exchange

```cpp
[[nodiscard]] auto exchange(std::uint64_t desired) noexcept -> std::uint64_t
{
    return value_.exchange(desired, std::memory_order_relaxed);
}
```

**行为**: 原子地将 `value_` 设为 `desired`，返回旧值。编译为 `lock xchg` 指令。

**用途**: 用于管理端重置计数器（`reset()`），或读取并清零。

## 使用场景

### 场景一：流量统计

[[core/stats/traffic|traffic_state]] 内部的每个原子字段本质上就是一个 `counter`（虽然 `traffic_state` 直接使用 `std::atomic` 而非 `counter` 类，但设计理念相同）：

```cpp
// traffic_state::flush_traffic 内部
total_uplink_.fetch_add(up, std::memory_order_relaxed);      // 递增上行字节
protocols_[i].uplink_bytes.fetch_add(up, std::memory_order_relaxed); // 协议维度
```

### 场景二：活跃连接计数

```cpp
// 伪代码：使用 counter 做连接计数
counter active_connections;

// 连接建立
active_connections.increment();

// 连接断开
active_connections.decrement();

// 读取当前值
auto current = active_connections.load();
```

### 场景三：读取并重置

```cpp
counter request_count;

// 累积计数...
request_count.increment();

// 定期采集：读取旧值并重置为 0
auto last_period = request_count.exchange(0);
// last_period 是上一个周期的增量，request_count 已归零
```

## 性能分析

### 单操作延迟

| 操作 | 指令 | 延迟（同核） | 延迟（跨核） |
|------|------|-------------|-------------|
| `increment()` | `lock xadd` | ~1ns | ~6ns |
| `decrement()` | `lock xadd` | ~1ns | ~6ns |
| `load()` | `mov` | ~0.3ns | ~0.3ns |
| `exchange()` | `lock xchg` | ~1ns | ~6ns |

### 缓存行开销

每个 `counter` 实例占用 64 字节内存，其中有效数据仅 8 字节，padding 占 56 字节。这是以空间换时间的经典做法：

```
假设 8 个 worker，每个 worker 有 10 个 counter：
  总内存 = 8 × 10 × 64B = 5120B = 5KB

如果不用 alignas(64)，10 个 counter 共享 1-2 个缓存行：
  总内存 = 8 × 10 × 8B = 640B
  但多核并发时性能退化 5-10x

权衡：5KB 内存（可忽略）换取 5-10x 性能提升（关键）
```

### 与 Traffic State 的对比

[[core/stats/traffic|traffic_state]] 内部的 `protocol_slot` 使用 `alignas(64)` 对齐整个结构体，但内部多个 `std::atomic` 共享同一缓存行：

```cpp
struct alignas(64) protocol_slot
{
    std::atomic<std::uint64_t> connections{0};   // offset 0
    std::atomic<std::uint64_t> active{0};        // offset 8
    std::atomic<std::uint64_t> uplink_bytes{0};  // offset 16
    std::atomic<std::uint64_t> downlink_bytes{0}; // offset 24
    // padding: offset 32-63
};
```

这是合理的，因为 `protocol_slot` 是 per-worker 单写者模式，内部字段之间不会产生跨核争用。只有不同 worker 的不同 `protocol_slot` 实例之间需要缓存行隔离，`alignas(64)` 保证了这一点。

## 与相关组件的对比

| 特性 | [[core/stats/counter\|counter]] | [[core/stats/gauge\|gauge]] | `std::atomic<uint64_t>` |
|------|------|------|------|
| 缓存行对齐 | 是 (alignas(64)) | 否 | 否 |
| 线程安全 | 是（原子操作） | 否（单写者） | 是（原子操作） |
| 数值类型 | `uint64_t` | `double` | 可配置 |
| 适用场景 | 连接计数、流量字节 | 延迟、负载百分比 | 灵活通用 |
| 内存开销 | 64 字节 | 16 字节 | 8 字节 |

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/gauge|gauge]] -- EMA 平滑瞬时值原语（互补原语）
- [[core/stats/traffic|traffic]] -- 使用原子计数器的流量统计
- [[core/stats/runtime|runtime]] -- 使用原子计数器的运行时统计
- [[core/instance/front/balancer|负载均衡器]] -- 消费 counter 数据做调度决策
