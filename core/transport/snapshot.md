---
layer: core
source: prism/transport/snapshot.hpp
module: "transport"
title: Snapshot 可回滚传输
tags: [transport, snapshot, 回滚, 装饰器, TLS伪装]
created: 2026-05-23
updated: 2026-05-28
---

# Snapshot 可回滚传输

> 命名空间: `psm::transport`
> 源码: `include/prism/transport/snapshot.hpp`

## 概述

`snapshot` 是可回滚的传输层装饰器，自动捕获所有从内层传输读取的字节到内部缓冲区，支持 `rewind()` 将读取位置归零重新读取。主要用于 TLS 伪装方案的依次尝试：每个 scheme 读取的数据被 snapshot 捕获，失败时 rewind，下一个 scheme 从同一起点重试。

## 设计约束

| 约束 | 说明 |
|------|------|
| rewind 仅在未发生写入时有效 | `wrote_ == false` |
| 认证阶段是纯读取 | 安全 rewind |
| 一旦开始写入 | transport 状态不可恢复 |

## 接口一览

| 方法 | 签名 | 说明 |
|------|------|------|
| 构造 | `snapshot(shared_transmission inner, resource_pointer mr)` | 包装内层传输 |
| `transport_type` | `() -> type` | 委托给内层传输 |
| `next_layer` | `() -> transmission *` | 返回内层传输指针 |
| `executor` | `() -> executor_type` | 委托给内层传输 |
| `async_read_some` | `(buffer, ec) -> awaitable<size_t>` | 两阶段读取：先从 captured_ 回放，再从内层读取并捕获 |
| `async_write_some` | `(buffer, ec) -> awaitable<size_t>` | 标记 wrote_=true 后委托给内层 |
| `rewind` | `() noexcept -> void` | 归零 read_pos_，不清空 captured_ |
| `can_rewind` | `() noexcept -> bool` | 返回 `!wrote_` |
| `close` / `cancel` | `() -> void` | 委托给内层传输 |
| `inner` | `() -> shared_transmission` | 返回内层传输 |

### 私有成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `inner_` | `shared_transmission` | 被包装的内层传输 |
| `captured_` | `memory::vector<std::byte>` | 捕获的读取数据 |
| `read_pos_` | `std::size_t` | 当前读取位置 |
| `wrote_` | `bool` | 是否已发生写入 |

### 工厂函数

`make_snapshot(shared_transmission inner, resource_pointer mr) -> shared_transmission` -- 创建 snapshot 包装器。

## 核心机制

### 两阶段读取

**Phase 1 (回放)**: 若 `read_pos_ < captured_.size()`，从 captured_ 复制数据到用户 buffer，同步完成，不挂起协程。

**Phase 2 (捕获)**: 从内层传输异步读取数据，同时将读取的数据追加到 captured_ 缓冲区，更新 read_pos_。

### 写入与 rewind 限制

`async_write_some` 在写入前设置 `wrote_ = true`，此后 `can_rewind()` 返回 `false`。原因：写入操作会改变传输层内部状态（如 TLS 握手状态），回滚无法恢复。

### rewind 操作

`rewind()` 仅将 `read_pos_` 归零，保留 captured_ 数据。下次 async_read_some 从 captured_ 起点回放所有已捕获数据。

## 使用场景

### TLS 伪装方案依次尝试

```
make_snapshot(inbound)
  -> Scheme A (reality): 读取数据，写入握手，失败 -> rewind()
  -> Scheme B (shadowtls): 从头读取，失败 -> rewind()
  -> Scheme C (restls): 从头读取，成功 -> 继续使用
```

每个方案从同一起点开始读取。仅当方案未执行写入时才能 rewind 到起点重试。方案成功后 snapshot 可被丢弃，释放 captured_ 内存。

### 与 preview 配合

先包装预读数据（`wrap_with_preview`），再包装 snapshot 用于方案回滚。preview 处理最初的 24 字节预读，snapshot 处理后续方案尝试。

## 性能特征

| 操作 | 开销 |
|------|------|
| 构造 | O(1) |
| Phase 1 读取 | O(N) memcpy |
| Phase 2 读取 | O(N) memcpy + 异步 I/O |
| 写入 | O(1) + 异步 I/O |
| rewind | O(1) |
| 内存占用 | O(总读取量)，通常几百字节到几 KB |

## 设计决策

### 为什么 rewind 只归零 read_pos_ 不清空 captured_

**问题**: TLS 伪装方案依次尝试时，每个 scheme 读取的数据需要被下一个 scheme 重用。

**选择**: rewind() 仅归零 read_pos_，captured_ 数据保留。下一个 scheme 从 captured_ 起点回放所有已捕获数据。

**后果**: 不需要重复从 socket 读取，数据仅在首次读取时产生真正的 I/O。代价是 captured_ 持续累积，内存占用随总读取量增长。

### 为什么写入后禁止 rewind

**问题**: 写入操作（如 TLS 握手的 ServerHello）会改变传输层内部状态（TCP 序列号、TLS 状态机），回滚无法恢复。

**选择**: async_write_some 在写入前设置 `wrote_ = true`，can_rewind() 返回 false。

**后果**: 方案切换只能在纯读取阶段进行。一旦开始写入，传输层状态不可逆。

### 为什么 snapshot 不支持 completion-handler 路径覆写

**问题**: snapshot 主要在 stealth 模块中使用，操作通过协程路径调用，不经过 ssl::stream 内部的 completion-handler 路径。

**选择**: 不覆写 completion-handler 路径，使用基类默认的 co_spawn 桥接。

**后果**: ssl::stream 包装在 snapshot 内部时 TLS 读写会多一个协程帧。但 snapshot 通常在 ssl::stream 创建之前使用，不受影响。

## 约束

### captured_ 无上限增长

**类型**: 资源消耗
**规则**: captured_ 随读取量持续累积，无容量限制。通常 TLS 握手阶段捕获几百字节到几 KB，方案成功后 snapshot 被丢弃释放。
**违反后果**: 协议异常导致大量读取而不关闭时，captured_ 可能消耗大量内存。

### rewind 前必须检查 can_rewind()

**类型**: 安全性
**规则**: 调用 rewind() 前应先检查 can_rewind()。wrote_==true 时 rewind() 会执行（归零 read_pos_），但回放的数据可能导致后续操作状态不一致。
**违反后果**: 在写入后 rewind 可能导致协议状态错误。

## 相关页面

- [[core/transport/overview|transport -- 传输层总览]]
- [[core/transport/preview|preview -- 预读数据回放]]
- [[core/transport/transmission|transmission -- 传输抽象]]
- [[core/stealth/overview|stealth -- TLS 伪装]]
- [[core/recognition|recognition -- 协议识别]]
- [[core/memory/overview|memory -- PMR 内存管理]]
