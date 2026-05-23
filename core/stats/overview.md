---
title: Stats 统计模块总览
layer: core
source:
  - I:/code/Prism/include/prism/stats/stats.hpp
  - I:/code/Prism/include/prism/stats/counter.hpp
  - I:/code/Prism/include/prism/stats/gauge.hpp
  - I:/code/Prism/include/prism/stats/snapshot.hpp
  - I:/code/Prism/include/prism/stats/runtime.hpp
  - I:/code/Prism/include/prism/stats/traffic.hpp
  - I:/code/Prism/include/prism/stats/account.hpp
module: stats
tags: [stats, module, architecture]
created: 2026-05-23
updated: 2026-05-23
---

# Stats 统计模块总览

Stats 模块是 Prism 的运行时统计基础设施，提供零分配、无锁的性能指标收集能力。该模块从原有 `instance::worker::stats` 演化而来，经过解耦和泛化，形成独立的统计原语层，被 [[core/instance/worker/worker|Worker]]、[[core/instance/front/balancer|负载均衡器]]、[[core/pipeline/overview|协议处理器]] 等模块广泛使用。

> 注意：[[core/instance/worker/stats|Worker 负载统计]]（`instance::worker::stats.hpp`）是旧版实现，stats 模块是新版独立基础设施。两者功能重叠，新版逐步替代旧版。

## 设计原则

- **零分配**: 所有热路径统计操作使用原子指令（`fetch_add`/`fetch_sub`），无动态内存分配
- **无锁并发**: 全部使用 `std::atomic` + `memory_order_relaxed`，避免互斥锁和内存屏障
- **缓存行对齐**: `counter` 和 `traffic_state` 使用 `alignas(64)` 独占缓存行，防止 false sharing
- **松散一致**: 快照是瞬时值的近似拷贝，适用于监控面板和日志，不保证跨字段原子性
- **单写者设计**: `traffic_state` 和 `gauge` 设计为 per-worker 单写者，减少跨核原子操作开销

## 模块组成

| 模块 | 功能 | 源码 |
|------|------|------|
| [[core/stats/stats|stats]] | 聚合头文件，引入所有子模块 | `prism/stats/stats.hpp` |
| [[core/stats/counter|counter]] | 缓存行对齐原子计数器原语 | `prism/stats/counter.hpp` |
| [[core/stats/gauge|gauge]] | EMA 平滑瞬时值原语 | `prism/stats/gauge.hpp` |
| [[core/stats/snapshot|snapshot]] | 所有快照类型定义 | `prism/stats/snapshot.hpp` |
| [[core/stats/runtime|runtime]] | Worker 负载监控 + 全局运行状态 | `prism/stats/runtime.hpp` |
| [[core/stats/traffic|traffic]] | Per-worker 流量统计 + 全局聚合 | `prism/stats/traffic.hpp` |
| [[core/stats/account|account]] | 账户统计观察者 | `prism/stats/account.hpp` |

## 架构层次

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Stats 模块层次结构                               │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                   聚合入口层 (stats.hpp)                       │  │
│  │  #include <prism/stats/stats.hpp>                             │  │
│  │  → 引入 counter, gauge, snapshot, runtime, traffic, account   │  │
│  └───────────────────────────┬───────────────────────────────────┘  │
│                              │                                      │
│  ┌───────────────────────────┼───────────────────────────────────┐  │
│  │                     业务统计层                                 │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐│  │
│  │  │   runtime    │  │   traffic    │  │      account         ││  │
│  │  │ worker_load  │  │traffic_state │  │  account_snapshot    ││  │
│  │  │ system_state │  │ protocol_slot│  │    collect()         ││  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘│  │
│  └─────────┼─────────────────┼─────────────────────┼─────────────┘  │
│            │                 │                     │                │
│  ┌─────────┼─────────────────┼─────────────────────┼─────────────┐  │
│  │         │          快照类型层 (snapshot.hpp)      │            │  │
│  │         │  worker_load_snapshot                  │            │  │
│  │         │  runtime_snapshot                      │            │  │
│  │         │  protocol_snapshot / traffic_snapshot  │            │  │
│  │         │                                        │            │  │
│  └─────────┼────────────────────────────────────────┼────────────┘  │
│            │                                                  │      │
│  ┌─────────┼──────────────────────────────────────────────────┐    │
│  │         │            原语层 (primitives)                    │    │
│  │  ┌──────────────┐  ┌──────────────┐                        │    │
│  │  │   counter    │  │    gauge     │                        │    │
│  │  │ atomic<uint> │  │  EMA 平滑    │                        │    │
│  │  │ alignas(64)  │  │  alpha=7/8   │                        │    │
│  │  └──────────────┘  └──────────────┘                        │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

## 数据流

### 统计写入路径（热路径）

```
新连接到达
    │
    ▼
launch::start()
    ├── traffic_state::on_connect()       ← total_connections++ + total_active++
    ├── traffic_state::on_auth_success()  ← 认证成功计数
    │   或 on_auth_failure()              ← 认证失败计数
    └── worker_load::session_open()       ← active_sessions++
    │
    ▼
session::diversion()
    └── traffic_state::on_protocol_detected(type)  ← protocol_slots[type].connections++ + active++
    │
    ▼
tunnel / udp_relay / mux core (数据传输阶段)
    └── 局部变量累积 uplink / downlink 字节
    │
    ▼
会话/子流结束
    ├── traffic_state::flush_traffic(proto, up, down)  ← 4 次 fetch_add 刷入
    ├── traffic_state::on_disconnect(type)              ← total_active-- + protocol_slots[type].active--
    └── worker_load::session_close()                    ← active_sessions--
```

### 统计读取路径（冷路径）

```
监控系统 / 管理接口
    │
    ├── traffic_state::aggregate()   ← 遍历 COW 注册表，汇总所有 worker
    │   └── 返回 traffic_snapshot
    │
    ├── worker_load::snapshot()      ← 读三个原子变量
    │   └── 返回 worker_load_snapshot
    │
    ├── system_state::snapshot()     ← 读启动时间 + worker 数量
    │   └── 返回 runtime_snapshot
    │
    └── account::collect(dir)        ← 遍历账户目录
        └── 返回 vector<account_snapshot>
```

## 线程安全模型

| 组件 | 写入者 | 读取者 | 安全机制 |
|------|--------|--------|----------|
| [[core/stats/counter\|counter]] | 多写者（`fetch_add`） | 多读者（`load`） | `std::atomic` + `relaxed` |
| [[core/stats/gauge\|gauge]] | 单写者 | 单读者 | 非线程安全，worker 内部使用 |
| [[core/stats/runtime\|worker_load]] | 单写者（worker 线程） | 多读者（balancer） | `std::atomic` + `relaxed` |
| [[core/stats/runtime\|system_state]] | 单次写入（main） | 多读者 | `std::atomic` + `exchange` |
| [[core/stats/traffic\|traffic_state]] | 单写者（worker 线程） | 多读者（aggregate） | `std::atomic` + `relaxed` |
| [[core/stats/traffic\|COW 注册表]] | worker 构造/析构时 | aggregate() 读取 | Copy-on-Write + acquire/release |
| [[core/stats/account\|account::collect]] | 无写入 | 调用者 | 纯观察者，只读 |

## 依赖关系

```
stats 模块的依赖图：

stats.hpp (聚合)
 ├── snapshot.hpp           ← 无外部依赖
 ├── counter.hpp            ← <atomic>, <cstdint>
 ├── gauge.hpp              ← 无外部依赖
 ├── runtime.hpp            ← <atomic>, <chrono>, <memory>, boost::asio, snapshot.hpp
 ├── traffic.hpp            ← <atomic>, <vector>, snapshot.hpp, protocol_type.hpp
 └── account.hpp            ← account/entry.hpp, account/directory.hpp, memory/container.hpp

stats 模块的上游消费者：

worker.hpp
 ├── stats::runtime::worker_load metrics_
 └── stats::traffic::traffic_state traffic_

launch.hpp
 └── stats::runtime::worker_load &metrics

balancer.hpp
 └── std::function<stats::worker_load_snapshot()> snapshot

context.hpp
 └── stats::traffic::traffic_state *traffic

protocol conn (trojan/vless/socks5)
 └── stats::traffic::traffic_state *traffic_

multiplex/core.hpp
 └── stats::traffic::traffic_state *traffic_
```

## 源码位置

> 源码: `include/prism/stats/` | 实现: `src/prism/stats/`

| 文件 | 行数 | 说明 |
|------|------|------|
| `stats.hpp` | 20 | 聚合头文件 |
| `counter.hpp` | 72 | 原子计数器 |
| `gauge.hpp` | 56 | EMA 仪表盘 |
| `snapshot.hpp` | 77 | 快照类型 |
| `runtime.hpp` | 125 | 运行时统计 |
| `traffic.hpp` | 153 | 流量统计 |
| `account.hpp` | 46 | 账户统计 |
| `runtime.cpp` | 147 | runtime 实现 |
| `traffic.cpp` | 179 | traffic 实现 |

## 相关文档

- [[core/stats/counter|counter]] -- 原子计数器原语详解
- [[core/stats/gauge|gauge]] -- EMA 平滑仪表盘详解
- [[core/stats/snapshot|snapshot]] -- 快照类型定义详解
- [[core/stats/runtime|runtime]] -- 运行时统计详解
- [[core/stats/traffic|traffic]] -- 流量统计详解
- [[core/stats/account|account]] -- 账户统计观察者详解
- [[core/instance/worker/worker|Worker 模块]] -- 统计的主要消费者
- [[core/instance/front/balancer|负载均衡器]] -- 使用 worker 负载快照做调度决策
