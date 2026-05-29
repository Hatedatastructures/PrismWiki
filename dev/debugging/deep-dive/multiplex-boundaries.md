---
title: 多路复用边界条件分析
source:
  - include/prism/multiplex/
  - src/prism/multiplex/
module: debugging
type: deep-dive
tags: [debugging, multiplex, smux, yamux, resource-limit, failure-analysis]
layer: dev
created: 2026-05-21
updated: 2026-05-21
related:
  - "[[core/multiplex/overview]]"
  - "[[core/multiplex/core]]"
---

# 多路复用边界条件分析

## 概述

多路复用系统的整体设计是健全的——流的生命周期管理、资源回收、背压机制均有明确保障。本文档仅记录在极端或异常场景下可观测到的边界条件，用于辅助调试与问题排查。

> **核心结论：不存在严重的流泄漏或资源无限增长路径。** 所有已知的边界条件均受到硬限制或超时机制的约束。

---

## Stream 生命周期

### 状态机

```
不存在 ──SYN──> pending ──activate──> active(duct/parcel) ──close──> 已关闭
```

- **duct**：对应 TCP 流，由 `Duct` 管理
- **parcel**：对应 UDP "流"，由 `Parcel` 管理

### 创建流程

1. 客户端发送 SYN 帧 → core 创建 pending 条目
2. 客户端发送 PSH/Data 帧，携带 SOCKS5 地址信息
3. core 调用 `activate_stream`，创建 duct 或 parcel
4. duct/parcel 连接目标地址，开始转发数据

### 关闭触发源

流关闭可由三种独立触发源发起：

| 触发源 | 路径 | 说明 |
|--------|------|------|
| 客户端 FIN | `handle_fin` → 设置 `mux_closed_` | 客户端主动关闭 |
| target EOF | target 读取返回 EOF → 设置 `target_closed_` | 远端关闭连接 |
| core 会话关闭 | `core::close()` → 取消所有流 | 整个 mux 会话终止 |

### 半关闭机制

每条流维护双标志：

- `mux_closed_`：mux 侧（客户端）已关闭
- `target_closed_`：target 侧已关闭

只有当**两端都关闭**时，才执行实际的 `close()` 操作。这确保了半关闭场景下数据仍能单向流动，直到对端也完成关闭。

`close()` 函数本身是**幂等**的——重复调用不会产生副作用。

---

## smux 边界条件

### pending 流无超时机制

smux 协议在创建 pending 流后，**没有设置超时定时器**。如果客户端发送 SYN 后不再发送足够的数据（SOCKS5 地址信息不完整），该 pending 条目将一直存在，直到整个 mux 会话关闭。

**缓解措施：**

- `max_streams = 32` 硬限制：单会话最多 32 条流，每个 pending 条目的缓冲区最大仅几十字节
- 会话本身有生命周期，不会永久存在

**影响评估：**

恶意客户端可以通过发送 SYN 后静默的方式占用 pending 槽位，但受限于 max_streams 上限，最多占用 32 个槽位。这不会导致内存无限增长，但可能暂时阻止合法流的创建。

---

## yamux 边界条件

### pending 超时自动清理

与 smux 不同，yamux 实现了 `start_pending_timeout()` 机制：

- pending 流创建时启动 **30 秒**超时定时器
- 超时后自动清理 pending 条目并向客户端发送 RST
- 这使得 yamux 在面对半初始化流时比 smux 更加健壮

### 窗口对象不泄漏

yamux 的 `windows_` 映射在所有清理路径上均会被正确移除：

- `remove_duct` / `remove_parcel`：流移除时清理
- `handle_fin` / `handle_rst`：协议层关闭时清理
- `yamux::close()`：会话级关闭时清理

每条路径都保证对应的 window 条目被删除，不存在泄漏路径。

---

## 资源上限

### 单会话资源占用上限

| 资源 | 上限 | 计算方式 |
|------|------|----------|
| 最大协程数 | **67** | 32 ducts x 2（读+写） + 控制流 |
| 内存上限 | **~66 MB** | 32 条 duct 的 `write_channel` 满载（极端条件） |
| 最大 fd 数 | **65** | 1 底层连接 + 32 target fd + 32 UDP socket |
| 并发流数 | **32** | `max_streams` 硬限制 |

### 流限制的强制执行

当活跃流数量达到 `max_streams` 时：

- 新的 SYN 帧会被**直接拒绝**
- 拒绝方式取决于协议实现（smux/yamux 各有处理）
- 不会创建新的 pending 条目，不会溢出限制

> 上述数值为单会话上限。实际部署中，多个会话的资源占用需要乘以并发会话数。

---

## 流控背压（yamux）

### 窗口机制

- **初始窗口大小**：256 KB
- 发送方在窗口不足时执行 `co_await window_signal->async_wait()`，暂停发送
- 接收方累积 consumed 数据量达到 `initial_window / 2` 时，发送 WindowUpdate 帧

### 异步分发设计

`dispatch_data` 使用 `co_spawn(detached)` 进行异步数据分发，**不阻塞帧循环**。这意味着：

- 帧循环可以持续接收和处理 WindowUpdate 帧
- 即使下游消费较慢，上游仍能处理控制帧
- **不存在死锁**：帧循环与 `send_data` 是独立的执行路径

### 背压传播路径

```
target 慢消费 → duct write_channel 满载 → co_await 暂停发送
→ 窗口不更新 → 客户端窗口耗尽 → 客户端暂停发送
```

整条链路的背压传播是完整的，不会出现数据堆积。

---

## 协程生命周期

### 保活机制

detached 协程通过持有 `shared_ptr<self>` 来确保自身不会被提前销毁。只要协程尚未完成，其所属的 duct/parcel 对象就不会被回收。

### 退出保证

协程退出的完整链路：

1. `core::close()` 被调用
2. 触发 `transport->cancel()`
3. frame_loop 中的读取操作被取消，frame_loop 退出
4. 触发 `channel_.cancel()`
5. send_loop 中的写入操作被取消，send_loop 退出
6. 所有 duct/parcel 的读写协程因取消信号而退出

**最大延迟**：一个心跳周期（30 秒）。在心跳超时之前，所有协程都能正常退出。

---

## 日志标签参考

调试时可关注以下日志标签：

| 标签 | 覆盖范围 |
|------|----------|
| `[Mux.Bootstrap]` | mux 会话的创建与引导 |
| `[Mux.Core]` | core 级别的流管理与协议分发 |
| `[Mux.Duct]` | TCP 流的创建、数据转发、关闭 |
| `[Mux.Parcel]` | UDP 流的创建、数据转发、关闭 |
| `[Smux.Craft]` | smux 协议帧的编解码 |
| `[Yamux.Craft]` | yamux 协议帧的编解码与窗口管理 |
