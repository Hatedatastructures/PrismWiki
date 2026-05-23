---
title: Account 账户统计观察者
layer: core
source: I:/code/Prism/include/prism/stats/account.hpp
module: stats
tags: [stats, account, observer]
created: 2026-05-23
updated: 2026-05-23
---

# Account 账户统计观察者

Account 子模块提供账户维度的统计快照收集能力。该模块是纯观察者模式——从 [[core/account/directory|account::directory]] 读取已有数据，不修改任何 [[core/account/entry|account::entry]] 的字段。stats 模块不依赖其他 stats 头文件，保持与统计基础设施的解耦。

> 源码: `include/prism/stats/account.hpp`

## 设计原则

- **纯观察者**: 只读操作，不修改 [[core/account/entry|account::entry]] 的任何字段
- **零依赖 stats 内部**: `account.hpp` 不 include 其他 stats 头文件（counter、gauge 等）
- **PMR 友好**: 使用 [[core/memory/container|memory::vector]] 返回结果，支持 PMR 内存资源
- **松散一致**: 快照是原子计数器的瞬时读取，不保证跨字段一致性

## account_snapshot 结构体

```cpp
struct account_snapshot
{
    memory::string credential;             // 账户凭证哈希
    std::uint64_t uplink_bytes{0};         // 上行总字节数
    std::uint64_t downlink_bytes{0};       // 下行总字节数
    std::uint32_t active_connections{0};   // 当前活跃连接数
    std::uint32_t max_connections{0};      // 最大允许连接数
};
```

### 字段详解

| 字段 | 类型 | 含义 | 来源 |
|------|------|------|------|
| `credential` | `memory::string` | 账户凭证哈希值 | [[core/account/entry\|account::entry]] 的标识字段 |
| `uplink_bytes` | `uint64_t` | 该账户的累计上行字节数 | entry 的原子计数器 |
| `downlink_bytes` | `uint64_t` | 该账户的累计下行字节数 | entry 的原子计数器 |
| `active_connections` | `uint32_t` | 该账户当前活跃的连接数 | entry 的原子计数器 |
| `max_connections` | `uint32_t` | 该账户的最大允许连接数 | entry 的配置字段 |

### 内存布局

```
sizeof(account_snapshot) ≈ 40+ 字节（取决于 memory::string 的 SSO 缓冲区大小）

┌────────────────────────────────────┐
│ credential  (memory::string)       │  ~24-32B (PMR string with SSO)
│ uplink_bytes    (uint64_t)         │  8B
│ downlink_bytes  (uint64_t)         │  8B
│ active_connections (uint32_t)      │  4B
│ max_connections    (uint32_t)      │  4B
│ padding                            │  4B (对齐)
└────────────────────────────────────┘
```

## collect() 函数

```cpp
[[nodiscard]] inline auto collect(
    const psm::account::directory &dir,
    memory::resource_pointer mr = memory::current_resource()
) -> memory::vector<account_snapshot>
{
    memory::vector<account_snapshot> result(mr);
    // TODO: 需要在 account::directory 中添加 for_each 遍历接口
    return result;
}
```

### 参数

| 参数 | 类型 | 含义 |
|------|------|------|
| `dir` | `const account::directory &` | 账户目录，COW 无锁读取 |
| `mr` | `memory::resource_pointer` | PMR 内存资源，默认使用当前线程的资源 |

### 返回值

`memory::vector<account_snapshot>` -- 所有账户的统计快照列表。

### 当前状态

> **注意**: `collect()` 当前为 TODO 桩实现。[[core/account/directory|account::directory]] 尚未暴露 `for_each` 遍历接口。完整实现需要：

1. `account::directory` 提供 `for_each` 或 `begin/end` 迭代器
2. 遍历每个 [[core/account/entry|account::entry]]
3. 对每个 entry 读取三个原子计数器的 `relaxed` 值
4. 构造 `account_snapshot` 并追加到结果列表

### 预期实现

```cpp
[[nodiscard]] inline auto collect(
    const psm::account::directory &dir,
    memory::resource_pointer mr = memory::current_resource()
) -> memory::vector<account_snapshot>
{
    memory::vector<account_snapshot> result(mr);
    dir.for_each([&result](const psm::account::entry &entry) {
        result.push_back(account_snapshot{
            .credential = memory::string(entry.credential(), result.get_allocator()),
            .uplink_bytes = entry.uplink_bytes.load(std::memory_order_relaxed),
            .downlink_bytes = entry.downlink_bytes.load(std::memory_order_relaxed),
            .active_connections = entry.active_connections.load(std::memory_order_relaxed),
            .max_connections = entry.max_connections
        });
    });
    return result;
}
```

## 与 account 模块的关系

```
┌────────────────────────────────────────────────────────────────┐
│                     account 模块                                │
│                                                                 │
│  ┌───────────────────┐    ┌──────────────────────────────────┐ │
│  │ account::entry    │    │ account::directory               │ │
│  │                   │    │                                   │ │
│  │ credential_       │    │ COW map<credential, entry>       │ │
│  │ atomic<uint64_t>  │    │ for_each() ← TODO               │ │
│  │   uplink_bytes_   │    └───────────┬──────────────────────┘ │
│  │ atomic<uint64_t>  │                │                        │
│  │   downlink_bytes_ │                │ collect() 读取         │
│  │ atomic<uint32_t>  │                │                        │
│  │   active_conns_   │                ▼                        │
│  │ uint32_t          │    ┌──────────────────────────────────┐ │
│  │   max_connections │    │ stats::account::collect()        │ │
│  └───────────────────┘    │                                   │ │
│                           │ 遍历 directory → 读取 entry       │ │
│                           │ → 构造 account_snapshot 列表      │ │
│                           └──────────────────────────────────┘ │
│                           │                                    │
│                           ▼                                    │
│                  ┌──────────────────────┐                      │
│                  │ vector<account_snap>  │                     │
│                  │ 返回给监控系统         │                     │
│                  └──────────────────────┘                      │
└────────────────────────────────────────────────────────────────┘
```

Stats 的 account 子模块是 account 模块的下游消费者。它不持有任何数据，仅在调用 `collect()` 时临时读取 account 模块的数据并构造快照。

## 依赖关系

```
stats::account 依赖:

prism/account/entry.hpp       ← account::entry 定义
prism/account/directory.hpp   ← account::directory 定义（COW 无锁读取）
prism/memory/container.hpp    ← memory::string, memory::vector

stats::account 的消费者（预期）:

管理接口 / 监控系统
    └── stats::account::collect(directory) → vector<account_snapshot>
        └── 序列化为 JSON 输出
```

## 使用场景

### 场景一：管理 API 查询

```cpp
// 查询所有账户的实时统计
auto snapshots = stats::account::collect(directory);
for (const auto &snap : snapshots)
{
    // 输出: credential=xxx, active=5/100, up=1.2GB, down=3.4GB
}
```

### 场景二：连接限制检查

```cpp
// 检查某账户是否超过最大连接数限制
auto snapshots = stats::account::collect(directory);
for (const auto &snap : snapshots)
{
    if (snap.active_connections >= snap.max_connections)
    {
        // 触发告警或拒绝新连接
    }
}
```

### 场景三：流量配额审计

```cpp
// 审计各账户的流量使用情况
auto snapshots = stats::account::collect(directory);
for (const auto &snap : snapshots)
{
    auto total = snap.uplink_bytes + snap.downlink_bytes;
    // 检查是否接近流量配额
}
```

## 线程安全

| 操作 | 线程安全 | 说明 |
|------|----------|------|
| `collect()` | 是 | 仅读取 `directory` 的 COW 数据和 `entry` 的原子计数器 |
| `account_snapshot` 构造 | 是 | 值类型，构造后不可变 |

`collect()` 的线程安全依赖于 [[core/account/directory|account::directory]] 的 COW 保证和 [[core/account/entry|account::entry]] 的原子计数器。调用者无需额外同步。

## 与 stats 模块其他组件的关系

`stats::account` 是 stats 模块中最独立的子模块：

| 依赖 | 说明 |
|------|------|
| 不依赖 [[core/stats/counter\|counter]] | 直接读取 entry 的 `std::atomic` |
| 不依赖 [[core/stats/gauge\|gauge]] | 账户统计不需要平滑 |
| 不依赖 [[core/stats/runtime\|runtime]] | 与 worker 负载无关 |
| 不依赖 [[core/stats/traffic\|traffic]] | 账户维度与协议维度正交 |
| 依赖 [[core/stats/snapshot\|snapshot]] | 仅概念上共享"快照"思想，无代码依赖 |

这种设计使得 `stats::account` 可以独立于其他 stats 组件使用和测试。

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/snapshot|snapshot]] -- 快照类型定义
- [[core/account/entry|account::entry]] -- 账户条目（数据来源）
- [[core/account/directory|account::directory]] -- 账户目录（COW 遍历）
- [[core/memory/container|memory::container]] -- PMR 容器定义
- [[core/instance/worker/launch|launch]] -- 账户认证入口
