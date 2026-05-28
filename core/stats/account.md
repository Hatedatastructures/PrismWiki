---
title: Account 账户统计观察者
layer: core
source: I:/code/Prism/include/prism/stats/account.hpp
module: stats
tags: [stats, account, observer]
created: 2026-05-23
updated: 2026-05-28
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

| 字段 | 类型 | 默认值 | 含义 | 来源 |
|------|------|--------|------|------|
| credential | memory::string | - | 账户凭证哈希值 | [[core/account/entry\|account::entry]] 的标识字段 |
| uplink_bytes | uint64_t | 0 | 该账户的累计上行字节数 | entry 的原子计数器 |
| downlink_bytes | uint64_t | 0 | 该账户的累计下行字节数 | entry 的原子计数器 |
| active_connections | uint32_t | 0 | 该账户当前活跃的连接数 | entry 的原子计数器 |
| max_connections | uint32_t | 0 | 该账户的最大允许连接数 | entry 的配置字段 |

## collect() 函数

| 项目 | 说明 |
|------|------|
| 签名 | `auto collect(const directory &dir, memory::resource_pointer mr) -> memory::vector<account_snapshot>` |
| 参数 dir | `const account::directory &` — 账户目录，COW 无锁读取 |
| 参数 mr | `memory::resource_pointer` — PMR 内存资源，默认使用当前线程的资源 |
| 返回值 | `memory::vector<account_snapshot>` — 所有账户的统计快照列表 |

### 当前状态

> **注意**: `collect()` 当前为 TODO 桩实现。[[core/account/directory|account::directory]] 尚未暴露 `for_each` 遍历接口。完整实现需要：

1. `account::directory` 提供 `for_each` 或 `begin/end` 迭代器
2. 遍历每个 [[core/account/entry|account::entry]]
3. 对每个 entry 读取三个原子计数器的 `relaxed` 值
4. 构造 `account_snapshot` 并追加到结果列表

## 依赖关系

### 上游依赖

| 依赖 | 说明 |
|------|------|
| `prism/account/entry.hpp` | account::entry 定义 |
| `prism/account/directory.hpp` | account::directory 定义（COW 无锁读取） |
| `prism/memory/container.hpp` | memory::string, memory::vector |

### 与 stats 模块其他组件的关系

`stats::account` 是 stats 模块中最独立的子模块：

| 依赖 | 说明 |
|------|------|
| 不依赖 [[core/stats/counter\|counter]] | 直接读取 entry 的 `std::atomic` |
| 不依赖 [[core/stats/gauge\|gauge]] | 账户统计不需要平滑 |
| 不依赖 [[core/stats/runtime\|runtime]] | 与 worker 负载无关 |
| 不依赖 [[core/stats/traffic\|traffic]] | 账户维度与协议维度正交 |
| 依赖 [[core/stats/snapshot\|snapshot]] | 仅概念上共享"快照"思想，无代码依赖 |

### 预期消费者

管理接口 / 监控系统调用 `collect(directory)` 获取快照列表，序列化为 JSON 输出。

## 线程安全

| 操作 | 线程安全 | 说明 |
|------|----------|------|
| `collect()` | 是 | 仅读取 directory 的 COW 数据和 entry 的原子计数器 |
| `account_snapshot` 构造 | 是 | 值类型，构造后不可变 |

`collect()` 的线程安全依赖于 [[core/account/directory|account::directory]] 的 COW 保证和 [[core/account/entry|account::entry]] 的原子计数器。调用者无需额外同步。

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/snapshot|snapshot]] -- 快照类型定义
- [[core/account/entry|account::entry]] -- 账户条目（数据来源）
- [[core/account/directory|account::directory]] -- 账户目录（COW 遍历）
- [[core/memory/container|memory::container]] -- PMR 容器定义
- [[core/instance/worker/launch|launch]] -- 账户认证入口
