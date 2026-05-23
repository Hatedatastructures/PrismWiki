---
title: Snapshot 统计快照类型
layer: core
source: I:/code/Prism/include/prism/stats/snapshot.hpp
module: stats
tags: [stats, snapshot, data-structure]
created: 2026-05-23
updated: 2026-05-23
---

# Snapshot 统计快照类型

`snapshot.hpp` 定义了 Stats 模块所有的只读快照类型。快照是对原子计数器瞬时值的松散一致拷贝，将线程安全的内部表示转换为可自由传递的值类型。所有业务统计方法（`snapshot()`、`aggregate()`）返回的都是此文件中定义的结构体。

> 源码: `include/prism/stats/snapshot.hpp`

## 设计原则

- **值类型**: 快照是 POD-like 结构体，可自由拷贝、移动，无需担心生命周期
- **松散一致**: 快照中各字段的读取不是原子操作，跨字段不保证一致性
- **监控友好**: 快照结构体可直接序列化为 JSON 或其他格式，输出到监控面板
- **零依赖**: 仅依赖 `<cstdint>`，不依赖 atomic、string 等重量级头文件

## 结构体一览

| 结构体 | 用途 | 生产者 |
|--------|------|--------|
| `worker_load_snapshot` | 单 worker 负载快照 | [[core/stats/runtime\|worker_load::snapshot()]] |
| `runtime_snapshot` | 全局运行状态快照 | [[core/stats/runtime\|system_state::snapshot()]] |
| `protocol_snapshot` | 单协议流量快照 | [[core/stats/traffic\|traffic_state]] 内部使用 |
| `traffic_snapshot` | 全局流量聚合快照 | [[core/stats/traffic\|traffic_state::aggregate()]] |

## 常量定义

```cpp
static constexpr std::size_t protocol_slot_count = 16;
```

协议槽位数组大小。当前 [[core/protocol/overview|protocol_type]] 枚举使用 0-8（unknown, http, socks5, trojan, vless, shadowsocks, tls, trusttunnel, anytls），预留 16 个槽位用于未来扩展（WebSocket、gRPC 等），无需改变内存布局。

## worker_load_snapshot

```cpp
struct worker_load_snapshot
{
    std::uint32_t active_sessions{0};       // 当前活跃会话数
    std::uint32_t pending_handoffs{0};      // 等待分发的连接数
    std::uint64_t event_loop_lag_us{0};     // 事件循环延迟（微秒，EMA 平滑后）
};
```

### 字段详解

| 字段 | 类型 | 单位 | 含义 |
|------|------|------|------|
| `active_sessions` | `uint32_t` | 个 | 当前正在处理的连接数量 |
| `pending_handoffs` | `uint32_t` | 个 | 已投递但尚未开始处理的连接数量 |
| `event_loop_lag_us` | `uint64_t` | 微秒 | 事件循环延迟，经 [[core/stats/gauge\|EMA]] 平滑后 |

### 生产者

[[core/stats/runtime|worker_load::snapshot()]]:

```cpp
auto worker_load::snapshot() const noexcept -> worker_load_snapshot
{
    return {
        active_sessions_->load(std::memory_order_relaxed),
        pending_handoffs_.load(std::memory_order_relaxed),
        event_loop_lag_us_.load(std::memory_order_relaxed)
    };
}
```

### 消费者

- [[core/instance/front/balancer|负载均衡器]]: 通过 `score()` 方法计算 worker 负载评分
- [[core/instance/worker/worker|Worker]]: 通过 `load_snapshot()` 方法暴露给外部

### 大小

```
sizeof(worker_load_snapshot) = 16 字节
  active_sessions:    4 字节
  pending_handoffs:   4 字节
  event_loop_lag_us:  8 字节
```

紧凑的 16 字节结构，适合高频采集和缓存行存储。

## runtime_snapshot

```cpp
struct runtime_snapshot
{
    std::uint64_t uptime_seconds{0};        // 进程运行时间（秒）
    std::uint32_t worker_count{0};           // 工作线程数量
};
```

### 字段详解

| 字段 | 类型 | 单位 | 含义 |
|------|------|------|------|
| `uptime_seconds` | `uint64_t` | 秒 | 自 `mark_started()` 调用以来的运行时间 |
| `worker_count` | `uint32_t` | 个 | 工作线程数量（通常为 CPU 核心数 - 1） |

### 生产者

[[core/stats/runtime|system_state::snapshot()]]:

```cpp
auto system_state::snapshot() const noexcept -> runtime_snapshot
{
    if (!started_.load(std::memory_order_relaxed))
        return {};
    const auto now = std::chrono::steady_clock::now();
    const auto uptime = std::chrono::duration_cast<std::chrono::seconds>(now - start_time_).count();
    return {static_cast<std::uint64_t>(uptime), worker_count_};
}
```

运行时间通过 `steady_clock` 计算，不受系统时间调整影响。

## protocol_snapshot

```cpp
struct protocol_snapshot
{
    std::uint64_t connections{0};            // 历史总连接数（仅增不减）
    std::uint64_t active{0};                 // 当前活跃连接数
    std::uint64_t uplink_bytes{0};           // 上行总字节数（含协议开销）
    std::uint64_t downlink_bytes{0};         // 下行总字节数（含协议开销）
};
```

### 字段详解

| 字段 | 类型 | 方向 | 含义 |
|------|------|------|------|
| `connections` | `uint64_t` | 仅增 | 历史总连接数，不随断开递减 |
| `active` | `uint64_t` | 增减 | 当前活跃连接数，建立时增，断开时减 |
| `uplink_bytes` | `uint64_t` | 仅增 | 客户端到上游方向的总字节数 |
| `downlink_bytes` | `uint64_t` | 仅增 | 上游到客户端方向的总字节数 |

### 协议维度索引

`protocol_snapshot` 通过 `traffic_snapshot::protocols[]` 数组按 [[core/protocol/overview|protocol_type]] 枚举值索引：

```
protocols[0] = unknown        → 未识别协议
protocols[1] = http           → HTTP 代理
protocols[2] = socks5         → SOCKS5 代理
protocols[3] = trojan         → Trojan 协议
protocols[4] = vless          → VLESS 协议
protocols[5] = shadowsocks    → Shadowsocks 2022
protocols[6] = tls            → TLS 协议
protocols[7] = trusttunnel    → TrustTunnel
protocols[8] = anytls         → AnyTLS
protocols[9-15] = reserved    → 预留扩展
```

## traffic_snapshot

```cpp
struct traffic_snapshot
{
    std::uint64_t total_connections{0};      // 全局总连接数
    std::uint64_t total_active{0};           // 全局当前活跃连接数
    std::uint64_t total_uplink{0};           // 全局上行总字节
    std::uint64_t total_downlink{0};         // 全局下行总字节
    protocol_snapshot protocols[protocol_slot_count]{};  // 按协议维度的流量明细
    std::uint64_t auth_success{0};           // 认证成功次数
    std::uint64_t auth_failure{0};           // 认证失败次数
};
```

### 字段详解

| 字段 | 类型 | 含义 |
|------|------|------|
| `total_connections` | `uint64_t` | 所有协议的历史总连接数 |
| `total_active` | `uint64_t` | 所有协议的当前活跃连接数 |
| `total_uplink` | `uint64_t` | 所有协议的上行字节汇总 |
| `total_downlink` | `uint64_t` | 所有协议的下行字节汇总 |
| `protocols[16]` | `protocol_snapshot[]` | 按协议类型索引的流量明细 |
| `auth_success` | `uint64_t` | 认证成功总次数 |
| `auth_failure` | `uint64_t` | 认证失败总次数 |

### 生产者

[[core/stats/traffic|traffic_state::aggregate()]] 遍历所有 worker 的 `traffic_state`，逐字段累加：

```cpp
auto traffic_state::aggregate() noexcept -> traffic_snapshot
{
    auto *reg = load_registry();
    if (!reg) return {};
    traffic_snapshot result;
    for (auto *instance : *reg)
    {
        auto s = instance->snapshot();
        result.total_connections += s.total_connections;
        result.total_active += s.total_active;
        result.total_uplink += s.total_uplink;
        result.total_downlink += s.total_downlink;
        result.auth_success += s.auth_success;
        result.auth_failure += s.auth_failure;
        for (std::size_t i = 0; i < protocol_slot_count; ++i)
        {
            result.protocols[i].connections += s.protocols[i].connections;
            result.protocols[i].active += s.protocols[i].active;
            result.protocols[i].uplink_bytes += s.protocols[i].uplink_bytes;
            result.protocols[i].downlink_bytes += s.protocols[i].downlink_bytes;
        }
    }
    return result;
}
```

### 大小

```
sizeof(traffic_snapshot)
  = 4 × 8                  (total 字段)
  + 16 × 4 × 8             (protocol_snapshot[16])
  + 2 × 8                  (auth 字段)
  = 32 + 512 + 16
  = 560 字节
```

560 字节的快照结构体。对于聚合操作（通常由管理接口调用，频率远低于热路径），这个大小完全可以接受。

## 松散一致性分析

快照的"松散一致"意味着：

```
时刻 T1: 读取 active_sessions = 100
时刻 T2: 读取 pending_handoffs = 5     ← T1 和 T2 之间可能有新连接
时刻 T3: 读取 event_loop_lag_us = 1200 ← T2 和 T3 之间 lag 可能变化

快照返回的是三个不同时刻的值的组合。
```

这对所有消费者都可接受：

| 消费者 | 需要的精度 | 松散一致的影响 |
|--------|-----------|---------------|
| [[core/instance/front/balancer\|Balancer]] | 近似值 | 评分偏差 < 1%，不影响调度正确性 |
| 监控面板 | 趋势值 | 秒级采样中微秒级偏差不可感知 |
| 日志 | 快照值 | 用于事后分析，不要求精确 |
| 告警 | 阈值判断 | 松散值在阈值附近可能有微小误差 |

## 序列化示例

```json
{
    "runtime": {
        "uptime_seconds": 86400,
        "worker_count": 7
    },
    "traffic": {
        "total_connections": 15234,
        "total_active": 312,
        "total_uplink": 107374182400,
        "total_downlink": 214748364800,
        "auth_success": 15100,
        "auth_failure": 134,
        "protocols": {
            "http": {"connections": 1000, "active": 20, "uplink_bytes": ..., "downlink_bytes": ...},
            "socks5": {"connections": 5000, "active": 100, ...},
            "trojan": {"connections": 8000, "active": 150, ...},
            "vless": {"connections": 1234, "active": 42, ...}
        }
    }
}
```

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/runtime|runtime]] -- worker_load_snapshot 和 runtime_snapshot 的生产者
- [[core/stats/traffic|traffic]] -- protocol_snapshot 和 traffic_snapshot 的生产者
- [[core/stats/counter|counter]] -- 快照字段的底层原子原语
- [[core/stats/gauge|gauge]] -- event_loop_lag_us 的 EMA 平滑原理
- [[core/instance/front/balancer|负载均衡器]] -- worker_load_snapshot 的主要消费者
