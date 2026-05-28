---
layer: core
source: include/prism/account/entry.hpp
title: Account Entry 与 Lease
module: account
tags:
  - entry
  - lease
  - RAII
  - atomic
  - traffic-stats
created: 2026-05-23
updated: 2026-05-27
---

# Account Entry 与 Lease

> 源码位置: `include/prism/account/entry.hpp`

## 概述

`entry.hpp` 定义了账户运行时状态结构和连接租约的 RAII 封装，是 [[core/account/overview|Account 模块]] 的核心数据结构。

## entry 结构体

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_connections` | `uint32_t` | 0 | 最大并发连接数，0 表示无限制 |
| `uplink_bytes` | `atomic_uint64_t` | 0 | 客户端→服务端方向累计流量 |
| `downlink_bytes` | `atomic_uint64_t` | 0 | 服务端→客户端方向累计流量 |
| `active_connections` | `atomic_uint32_t` | 0 | 当前活跃的连接数量 |

所有原子字段使用 `std::memory_order_relaxed` — 流量统计和连接计数仅用于近似监控，不参与同步控制。relaxed 序在 x86 上几乎零开销。

`entry` 通过 `shared_ptr` 管理生命周期，多个 lease 可共享同一 entry（多协议凭证共享连接配额）。

## lease 类

| 方法 | 说明 |
|------|------|
| `lease(shared_ptr<entry>)` | 接管所有权，**不自动递增**连接数 |
| `lease(lease&&)` / `operator=(lease&&)` | 移动语义，移动后源对象为空 |
| `~lease()` | 调用 `release()` 递减连接数 |
| `get()` | 返回底层 `entry*`，空租约返回 `nullptr` |
| `explicit operator bool()` | 检查是否有效 |

禁止拷贝，防止重复递减。

## 自由函数

| 函数 | 说明 |
|------|------|
| `accumulate_uplink(entry*, bytes)` | relaxed 原子递增上行流量，空指针空操作 |
| `accumulate_downlink(entry*, bytes)` | relaxed 原子递增下行流量，空指针空操作 |

## 设计决策

### 为什么 lease 构造函数不自动递增？

`try_acquire()` 中 CAS 递增和 lease 构造必须是原子的：先 CAS 递增 `active_connections`，成功后才构造 lease。如果 lease 构造时递增，CAS 成功后、构造前的间隙可能导致计数不一致（CAS 已递增但 lease 未构造，异常路径漏减）。

**后果**: 调用方必须保证在构造 lease 前已完成递增。`try_acquire()` 封装了这个约定。

### 为什么 lease 是 RAII 而非手动 release？

连接异常退出（RST、超时、协程取消）时，如果忘记调用 `release()`，`active_connections` 永远不递减，最终达到上限拒绝所有新连接。RAII 析构函数保证无论退出路径如何都递减计数。

**后果**: lease 不可拷贝（防止重复递减），只可移动。

### 为什么使用 relaxed 内存序？

`uplink_bytes`、`downlink_bytes` 和 `active_connections` 仅用于近似统计和连接限制：
- 流量统计用于监控和日志，精确性要求低
- 连接限制是"尽力而为"的软限制，少量误差可接受
- relaxed 在 x86 上等同于普通读写，零额外开销

**后果**: 如果需要精确的统计快照（读取时所有字段一致），需要外部同步机制。

## 约束

### lease 必须在同一线程析构

**类型**: 线程安全

**规则**: lease 不可跨线程传递。虽然 `fetch_sub` 本身线程安全，但移动语义（`shared_ptr` 的 move）不是线程安全的。

**违反后果**: 重复递减或漏递减。

**源码依据**: `entry.hpp:70-73` 移动构造函数

### 移动后的源 lease 不可使用

**类型**: 状态前置

**规则**: 移动构造/赋值后，源 lease 的 `state_` 为 `nullptr`，调用 `get()` 返回 `nullptr`，`operator bool()` 返回 `false`。

**违反后果**: 对已移动的 lease 调用 `get()` 返回 `nullptr`，后续 `accumulate_*` 函数空指针空操作，流量统计丢失但不崩溃。

**源码依据**: `entry.hpp:70-73`

## 使用场景

| 场景 | 调用方 | 说明 |
|------|--------|------|
| 连接认证 | `try_acquire(dir, cred)` | CAS 递增 + 构造 lease |
| 流量统计 | `accumulate_uplink/downlink(lease.get(), n)` | 每次 tunnel 转发后调用 |
| 所有权转移 | `handle_session(std::move(lease))` | 协程间移动 |
| 连接释放 | `~lease()` | 协程退出自动析构 |

## 引用关系

### 依赖

- （无外部依赖，仅标准库 `<atomic>`, `<memory>`）

### 被引用

- [[core/account/directory|directory]]：通过 `try_acquire()` 创建 lease
- [[core/context/context|Context]]：session 持有 `account::lease` 字段
