---
title: Traffic 流量统计
layer: core
source:
  - I:/code/Prism/include/prism/stats/traffic.hpp
  - I:/code/Prism/src/prism/stats/traffic.cpp
module: stats
tags: [stats, traffic, protocol, cow, performance]
updated: 2026-05-27
---

# Traffic 流量统计

Per-worker 流量统计和全局聚合。每个 [[core/instance/worker/worker|Worker]] 持有一个 `traffic_state` 实例，热路径中批量刷入流量数据。全局聚合使用 COW 注册表。

## 核心组件

### protocol_slot

`alignas(64)` 的原子计数器组，每个协议一个 slot：

| 字段 | 方向 | 含义 |
|------|------|------|
| `connections` | 仅增 | 历史总连接数 |
| `active` | 增减 | 当前活跃连接数 |
| `uplink_bytes` | 仅增 | 上行累计字节数 |
| `downlink_bytes` | 仅增 | 下行累计字节数 |

### traffic_state

Per-worker 流量计数器，`alignas(64)` 独占缓存行。

| 方法 | 说明 | 原子操作数 |
|------|------|-----------|
| `on_connect()` | 新连接建立 | 2 fetch_add |
| `on_protocol_detected(type)` | 协议识别完成 | 2 fetch_add |
| `on_disconnect(type)` | 连接断开 | 2 fetch_sub |
| `flush_traffic(proto, up, down)` | 批量刷入流量 | 最多 4 fetch_add |
| `on_auth_success()` | 认证成功 | 1 fetch_add |
| `on_auth_failure()` | 认证失败 | 1 fetch_add |
| `snapshot()` | 获取松散一致快照 | 70 load |
| `aggregate()` | 全局聚合（static） | N x 140 操作 |

## 设计决策

### 为什么流量用批量刷入而非逐字节？

热路径中流量在局部变量累积，会话/子流结束时一次性调用 `flush_traffic()`。每次会话最多 10 次原子操作（建立 4 + 刷入 4 + 断开 2），相比逐字节 fetch_add 的数千次操作，开销从 O(bytes) 降到 O(1)。

**后果**: `flush_traffic()` 的 `up`/`down` 参数在调用前累积在局部变量中，会话中途崩溃时这部分流量丢失。

### 为什么 COW 注册表不释放旧 vector？

`aggregate()` 可能正在遍历旧注册表。如果释放旧 vector，遍历中的指针变为悬挂。COW 替换后旧 vector 泄漏（约 24 + 8*N 字节），但 worker 数量通常 < 64，泄漏总量 < 10KB。

**后果**: 每次 worker 创建/销毁都有少量内存泄漏。进程生命周期内可忽略。

### 为什么整个 traffic_state 而非逐字段 alignas(64)？

`traffic_state` 是 per-worker 单写者设计。内部多个原子字段共享缓存行不会导致 false sharing（同一 worker 只有一个写者）。整个类 `alignas(64)` 防止与其他 worker 的数据共享缓存行即可。

**后果**: `protocol_slot` 内部 4 个字段共享 64 字节缓存行，安全（单写者保证）。

## 约束

### flush_traffic 前必须累积在局部变量

**类型**: 调用顺序

**规则**: `flush_traffic()` 应在 tunnel/relay/mux stream 结束时调用，而非每次读写时调用。

**违反后果**: 每次 read/write 都调用 `flush_traffic()`，原子操作数从 O(1)/会话暴涨到 O(bytes)/会话，性能严重退化。

**源码依据**: `traffic.hpp:86` 注释

### unregister_instance 当前未调用

**类型**: 调用顺序

**规则**: `unregister_instance()` 存在但当前未使用。worker 生命周期等于进程生命周期，不需要注销。

**违反后果**: 如果中途销毁 worker 而不调用 `unregister_instance()`，`aggregate()` 会遍历到已析构的指针，未定义行为。

## 引用关系

### 被引用

- [[core/instance/worker/launch|launch::start()]]：调用 `on_connect()`, `on_auth_success/failure()`
- [[core/connect/tunnel/tunnel|tunnel]]：转发结束时调用 `flush_traffic()`
- [[core/multiplex/core|mux core]]：子流结束时调用 `flush_traffic()`
