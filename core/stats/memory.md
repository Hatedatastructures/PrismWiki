---
title: Memory Tracker 内存分配统计追踪器
created: 2026-05-17
updated: 2026-05-27
layer: core
source: include/prism/stats/memory.hpp
  - include/prism/stats/memory.hpp
tags: [stats, memory, tracker, pmr, atomic, instrumented, singleton, allocation]
---

# Memory Tracker 内存分配统计追踪器

`memory_tracker` 是全局单例的 PMR 池分配统计追踪器，通过 instrumented memory resource 在每次分配/释放时更新原子计数器，提供零分配、无锁的内存使用监控能力。

> 源码: `include/prism/stats/memory.hpp` | 依赖: [[core/stats/snapshot|snapshot.hpp]]

## 类定义

```cpp
namespace psm::stats
{
    class alignas(64) memory_tracker
    {
    public:
        [[nodiscard]] static auto instance() -> memory_tracker &;

        void on_allocate(std::uint64_t bytes) noexcept;
        void on_deallocate(std::uint64_t bytes) noexcept;

        [[nodiscard]] auto snapshot() const noexcept -> memory_snapshot;

        [[nodiscard]] auto total_allocated() const noexcept -> std::uint64_t;
        [[nodiscard]] auto current_usage() const noexcept -> std::uint64_t;
        [[nodiscard]] auto allocation_count() const noexcept -> std::uint64_t;

    private:
        std::atomic<std::uint64_t> total_allocated_{0};
        std::atomic<std::uint64_t> total_deallocated_{0};
        std::atomic<std::uint64_t> current_usage_{0};
        std::atomic<std::uint64_t> allocation_count_{0};
    };
}
```

## 设计决策

### alignas(64) 缓存行对齐

与 [[core/stats/counter|counter]] 相同的设计理念：独占缓存行，防止 false sharing。`memory_tracker` 作为全局单例，虽然只有一个实例，但对齐确保其内部 `atomic` 字段不与相邻内存对象的修改产生缓存行争用。

### memory_order_relaxed

所有原子操作使用 `memory_order_relaxed`，理由：

- `on_allocate()` / `on_deallocate()` 仅递增/递减计数器，不保护关联数据
- `snapshot()` 返回松散一致快照，适用于监控面板和日志
- 不需要与其他变量建立 happens-before 关系

### 全局单例（Meyers' Singleton）

```cpp
static auto instance() -> memory_tracker &
{
    static memory_tracker inst;
    return inst;
}
```

C++11 保证线程安全的局部静态变量初始化。首次调用 `instance()` 时构造，进程生命周期内持续存在。

## 方法详解

### on_allocate

```cpp
void on_allocate(std::uint64_t bytes) noexcept
{
    total_allocated_.fetch_add(bytes, std::memory_order_relaxed);
    current_usage_.fetch_add(bytes, std::memory_order_relaxed);
    allocation_count_.fetch_add(1, std::memory_order_relaxed);
}
```

**行为**: 在 instrumented memory resource 的 `do_allocate()` 中调用，记录一次内存分配。

**原子操作**: 3 次 `fetch_add`（约 18ns）。

**语义**:
- `total_allocated_` — 累计分配字节数（仅增不减）
- `current_usage_` — 当前活跃字节数（增减平衡）
- `allocation_count_` — 累计分配次数（仅增不减）

### on_deallocate

```cpp
void on_deallocate(std::uint64_t bytes) noexcept
{
    total_deallocated_.fetch_add(bytes, std::memory_order_relaxed);
    current_usage_.fetch_sub(bytes, std::memory_order_relaxed);
}
```

**行为**: 在 instrumented memory resource 的 `do_deallocate()` 中调用，记录一次内存释放。

**原子操作**: 2 次（`fetch_add` + `fetch_sub`，约 12ns）。

### snapshot

```cpp
auto snapshot() const noexcept -> memory_snapshot
{
    return memory_snapshot{
        .total_allocated = total_allocated_.load(std::memory_order_relaxed),
        .total_deallocated = total_deallocated_.load(std::memory_order_relaxed),
        .current_usage = current_usage_.load(std::memory_order_relaxed),
        .allocation_count = allocation_count_.load(std::memory_order_relaxed),
    };
}
```

**行为**: 返回 [[core/stats/snapshot|memory_snapshot]] 结构体，包含四个字段的瞬时值。

**松散一致性**: 四个字段的读取不保证跨字段原子性（例如 `total_allocated - total_deallocated` 可能不精确等于 `current_usage`），但对监控场景足够。

### total_allocated / current_usage / allocation_count

单字段读取方法，各执行一次 `atomic load`。适合在日志或调试中快速检查单个指标。

## memory_snapshot 结构体

定义于 [[core/stats/snapshot|snapshot.hpp]]:

```cpp
struct memory_snapshot
{
    std::uint64_t total_allocated{0};     // 累计分配字节
    std::uint64_t total_deallocated{0};   // 累计释放字节
    std::uint64_t current_usage{0};       // 当前活跃字节
    std::uint64_t allocation_count{0};    // 分配次数
};
```

## 集成方式

`memory_tracker` 需要通过 instrumented memory resource 与 PMR 池集成：

```
业务代码调用 memory::vector::push_back()
  │
  └→ polymorphic_allocator::allocate()
       │
       └→ instrumented_resource::do_allocate()
            │
            ├→ memory_tracker::instance().on_allocate(bytes)
            └→ upstream_resource->allocate(bytes)
```

instrumented_resource 是包装层，在实际内存资源之上添加统计计数。`memory_tracker` 本身不直接管理内存，仅被动接收通知。

## 性能影响

| 操作 | 原子操作数 | 估计耗时 |
|------|-----------|---------|
| `on_allocate()` | 3 fetch_add | ~18ns |
| `on_deallocate()` | 1 fetch_add + 1 fetch_sub | ~12ns |
| `snapshot()` | 4 load | ~2ns |

相对于 PMR 池分配本身的 3-5ns 开销，追踪器增加了约 3-4 倍的开销。因此建议仅在开发/调试阶段启用，生产环境可通过条件编译关闭。

## 与 stats 模块其他组件的关系

| 组件 | 关系 |
|------|------|
| [[core/stats/snapshot|snapshot]] | memory_snapshot 类型定义 |
| [[core/stats/counter|counter]] | 相同的 alignas(64) + relaxed 设计 |
| [[core/stats/runtime|runtime]] | 互补的运行时统计（worker 负载） |
| [[core/stats/traffic|traffic]] | 互补的流量统计（连接/字节） |
| [[core/memory/pool|memory::pool]] | 被追踪的内存池 |

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/snapshot|snapshot]] -- memory_snapshot 快照类型
- [[core/stats/counter|counter]] -- 底层原子计数器原语
- [[core/memory/overview|Memory 模块]] -- PMR 内存管理基础设施
- [[core/memory/pool|Memory Pool]] -- 内存池系统
