---
title: Stats 统计模块入口
layer: core
source: I:/code/Prism/include/prism/stats/stats.hpp
module: stats
tags: [stats, module]
created: 2026-05-23
updated: 2026-05-23
---

# Stats 统计模块入口

`stats.hpp` 是统计模块的聚合头文件，提供一站式 `#include`。引入该头文件即可使用统计模块的全部功能。

> 源码: `include/prism/stats/stats.hpp`

## 头文件内容

```cpp
/**
 * @file stats.hpp
 * @brief 统计模块聚合头文件
 */
#pragma once

#include <prism/stats/snapshot.hpp>
#include <prism/stats/counter.hpp>
#include <prism/stats/gauge.hpp>
#include <prism/stats/runtime.hpp>
#include <prism/stats/traffic.hpp>
#include <prism/stats/account.hpp>
```

## 引入顺序

引入顺序反映模块的依赖层次：底层原语在前，业务模块在后。

| 引入 | 层次 | 说明 |
|------|------|------|
| [[core/stats/snapshot\|snapshot.hpp]] | 快照类型 | 所有业务模块的返回值类型定义 |
| [[core/stats/counter\|counter.hpp]] | 原语层 | 缓存行对齐原子计数器 |
| [[core/stats/gauge\|gauge.hpp]] | 原语层 | EMA 平滑瞬时值 |
| [[core/stats/runtime\|runtime.hpp]] | 业务层 | Worker 负载 + 全局运行状态 |
| [[core/stats/traffic\|traffic.hpp]] | 业务层 | Per-worker 流量 + 全局聚合 |
| [[core/stats/account\|account.hpp]] | 业务层 | 账户统计观察者 |

## 使用方式

### 全量引入

需要使用统计模块的多种功能时，引入聚合头文件：

```cpp
#include <prism/stats/stats.hpp>

// 可使用所有统计类型
psm::stats::counter conn_count;
psm::stats::gauge latency_gauge;
auto snap = traffic_state::aggregate();
```

### 选择性引入

仅需部分功能时，可单独引入对应子头文件以减少编译依赖：

```cpp
// 仅需计数器
#include <prism/stats/counter.hpp>
psm::stats::counter active_connections;

// 仅需快照类型
#include <prism/stats/snapshot.hpp>
psm::stats::traffic_snapshot snap;
```

## 模块结构映射

```
psm::stats 命名空间
 ├── counter                 ← counter.hpp
 ├── gauge                   ← gauge.hpp
 ├── worker_load_snapshot    ← snapshot.hpp
 ├── runtime_snapshot        ← snapshot.hpp
 ├── protocol_snapshot       ← snapshot.hpp
 ├── traffic_snapshot        ← snapshot.hpp
 ├── protocol_slot_count     ← snapshot.hpp
 ├── runtime::               ← runtime.hpp
 │   ├── worker_load
 │   └── system_state
 ├── traffic::               ← traffic.hpp
 │   ├── protocol_slot
 │   └── traffic_state
 └── account::               ← account.hpp
     ├── account_snapshot
     └── collect()
```

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构和设计原则
- [[core/stats/counter|counter]] -- 原子计数器原语
- [[core/stats/gauge|gauge]] -- EMA 平滑仪表盘
- [[core/stats/snapshot|snapshot]] -- 快照类型定义
- [[core/stats/runtime|runtime]] -- 运行时统计
- [[core/stats/traffic|traffic]] -- 流量统计
- [[core/stats/account|account]] -- 账户统计观察者
