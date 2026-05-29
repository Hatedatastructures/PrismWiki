---
title: tunnel
layer: core
source: include/prism/connect/tunnel/tunnel.hpp
module: connect
tags:
  - tunnel
  - relay
  - bidirectional
  - data-transfer
  - traffic-stats
created: 2026-05-23
updated: 2026-05-23
---

# tunnel

双向透明数据转发隧道，在入站和出站传输流之间建立全双工数据通道，是代理服务器的核心数据转发组件。

## 概述

`tunnel()` 函数是所有协议 TCP 隧道转发的最终汇聚点。无论连接经过 SOCKS5、Trojan、VLESS 还是 HTTP 协议处理，最终的数据转发都通过 `tunnel()` 完成。

核心特性：

- **全双工**：入站到出站和出站到入站两个方向同时转发
- **透明**：不修改数据内容，原样转发
- **协程并发**：使用 `awaitable_operators::operator||` 并发执行两个方向
- **流量统计**：记录上传/下载字节数，刷写到统计模块和账户用量
- **优雅关闭**：隧道结束后先半关闭（shutdown write）再关闭连接

## 函数签名

```cpp
auto tunnel(tunnel_options opts) -> net::awaitable<void>;
```

### tunnel_options

```cpp
struct tunnel_options
{
    shared_transmission inbound;                    // 入站流对象
    shared_transmission outbound;                   // 出站流对象
    const context::session &ctx;                    // 会话上下文
    write_policy policy{write_policy::complete};    // 写入策略
};
```

| 字段 | 说明 |
|------|------|
| `inbound` | 入站传输对象（客户端连接） |
| `outbound` | 出站传输对象（上游连接） |
| `ctx` | 会话上下文，提供缓冲区配置和统计信息 |
| `policy` | `complete` = `async_write()` 循环直到写完；`partial` = `async_write_some()` 单次写入 |

### write_policy 枚举

```cpp
enum class write_policy : std::uint8_t
{
    partial,   // async_write_some
    complete   // async_write（默认）
};
```

## 内部实现

### 缓冲区分配

从 `thread_local_pool()` 分配一块连续内存，前半给上传方向，后半给下载方向。缓冲区大小至少 2 字节（`max(buffer_size, 2)`），使用 PMR 分配器。

### 方向标识

| idx | 方向 | from | to | 含义 |
|-----|------|------|----|------|
| 0 | upload | inbound | outbound | 客户端 -> 上游 |
| 1 | download | outbound | inbound | 上游 -> 客户端 |

### 数据转发循环

每个方向在循环中执行：`async_read_some` 读取数据 → 累计字节 → 根据 `write_policy` 写入目标端。读返回 0 或错误时退出循环。

### 并发执行

使用 `boost::asio::experimental::awaitable_operators::operator||` 同时启动两个方向的转发协程。任一方向完成（读返回 0 或错误），`operator||` 立即返回，另一方向的协程被自动取消。

## 流量统计

隧道结束后，将上传/下载字节数刷写到 worker 级别的流量统计模块（`ctx.worker_ctx.traffic->flush_traffic()`），并累加到账户用量（`account::accumulate_uplink/downlink()`）。

**统计字段：**
- `total_bytes[0]`：上传字节数（inbound → outbound）
- `total_bytes[1]`：下载字节数（outbound → inbound）
- `ctx.detected_protocol`：识别出的协议类型

## 设计决策

### 为什么用单一分配双缓冲区？

从 `thread_local_pool()` 分配一块连续内存后分割为上下两个半区，比分别分配两个缓冲区少一次分配操作。上下行流量通常对称，共享同一块内存也改善了缓存局部性。

**后果**: 缓冲区大小必须是偶数（`max(buffer_size, 2)`），否则两半不等长。

### 为什么用 operator|| 而非两个独立 co_spawn？

`operator||` 在任一协程完成时自动取消另一个，无需手动管理取消逻辑。如果用 `co_spawn` 分别启动，需要额外的 cancellation_slot 和信号传递来协调关闭时机。

**后果**: 两个方向的转发协程必须在同一个 `io_context` 上运行（`operator||` 的要求）。

### 为什么任一方向断开即终止？

TCP 是全双工的，但代理场景中通常不需要半关闭语义。如果上游关闭连接，客户端应该立即感知到而非等待本地缓冲区排空。立即终止简化了状态管理，避免半关闭后的死锁风险。

**后果**: 客户端可能在本地发送缓冲区中还有未发送数据时被关闭。对大多数协议（HTTP/SOCKS5/Trojan）这是可接受的。

## 约束

### ctx 引用生命周期

**类型**: 生命周期

**规则**: `tunnel_options` 持有 `const context::session &`，session 必须在隧道完成前存活。

**违反后果**: 协程恢复后访问已释放的 session，悬挂引用。

**源码依据**: `tunnel.hpp:42`

### 缓冲区最小值

**类型**: 资源上限

**规则**: `ctx.buffer_size` 最小值为 2。小于 2 时被强制提升为 2（每个方向至少 1 字节）。

**违反后果**: 缓冲区为 0 时 `forward_data` 的 `async_read_some` 使用空 span，行为未定义。

**源码依据**: `tunnel.hpp:52`

## 连接关闭

```cpp
shut_close(inbound);
shut_close(outbound);
```

隧道结束后通过 [[core/connect/util|shut_close()]] 优雅关闭两个传输对象：

```
shut_close(trans)
    |
    v
[trans == nullptr?]
    |-- Yes --> 无操作
    |-- No  --> [lowest_layer<reliable>()?]
                    |-- 存在 --> shutdown_write()（半关闭）
                    +-- close()（关闭连接）
                    +-- reset()（释放 shared_ptr）
```

## 时序图

```
Client          tunnel()         inbound          outbound
  |                 |                |                |
  |---data--------->|                |                |
  |                 |--read_some()-->|                |
  |                 |<--data--------|                |
  |                 |                |                |
  |                 |--write_some()------------------>|
  |                 |                |                |
  |                 |                |--read_some()-->|
  |                 |                |<--data---------|
  |                 |<--data-------------------------|
  |                 |                |                |
  |<--data----------|                |                |
  |                 |                |                |
  | ... (concurrent bidirectional)  |                |
  |                 |                |                |
  |---close-------->|                |                |
  |                 |--read returns 0/err             |
  |                 |                |                |
  |                 |  operator|| completes           |
  |                 |                |                |
  |                 |--shut_close(inbound)------------|
  |                 |--shut_close(outbound)-----------|
  |                 |                |                |
  |                 |--flush_traffic()                |
  |                 |--accumulate_uplink/downlink()   |
  |                 |--log transfer stats             |
  |                 |                |                |
```

## 数据流图

```
                 tunnel()
                    |
    +---------------+---------------+
    |                               |
 forward_data(0)              forward_data(1)
 [upload]                     [download]
    |                               |
 inbound                    outbound
    |                               |
 async_read_some()           async_read_some()
    |                               |
    v                               v
 [left buffer]               [right buffer]
    |                               |
    v                               v
 outbound                   inbound
    |                               |
 async_write/                async_write/
 async_write_some            async_write_some
    |                               |
    v                               v
 [Upstream Server]           [Client]
```

## 调用链

```
[[core/connect/tunnel/forward|forward()]]
    |
    +-- dial() --> 获取出站传输
    |
    v
tunnel(inbound, outbound, ctx)
    |
    +-- 分配双缓冲区 (PMR)
    |
    +-- forward_data({inbound, outbound, left, 0})   // upload
    |       |
    |       +-- loop: async_read_some -> async_write
    |
    +-- forward_data({outbound, inbound, right, 1})  // download
    |       |
    |       +-- loop: async_read_some -> async_write
    |
    +-- operator|| 并发执行
    |
    v
[任一方向断开]
    |
    +-- shut_close(inbound)
    +-- shut_close(outbound)
    +-- flush_traffic()
    +-- accumulate_uplink/downlink()
    +-- log stats
```

## 性能特性

### 零拷贝策略

`tunnel()` 的缓冲区策略：

- 单次 PMR 分配，分配为一块连续内存
- 分割为两个半区，分别用于上传和下载
- 数据直接在内核缓冲区和用户缓冲区之间传递，不做额外拷贝
- `async_read_some` / `async_write_some` 由内核/SSL 层直接操作用户缓冲区

### 缓冲区大小影响

缓冲区大小（`ctx.buffer_size`）直接影响吞吐量：

| 缓冲区大小 | 每次 syscall 数据量 | 适用场景 |
|-----------|-------------------|---------|
| 4KB | 小 | 内存受限 |
| 16KB | 中 | 低带宽 |
| 32KB | 中高 | 一般场景 |
| 64KB（默认） | 高 | 高吞吐量 |
| 128KB+ | 很高 | 万兆网络 |

### 协程开销

- 两个方向的转发使用 `operator||` 并发执行，共享同一个线程
- 每次读/写都是一个 `co_await` 挂起点
- 挂起/恢复开销极低（无锁、无系统调用）

## 与其他模块的关系

- [[core/connect/tunnel/forward]]：`forward()` 组合 `dial()` + `tunnel()`，是 tunnel 的主要调用方
- [[core/connect/dial/dial]]：拨号函数提供出站传输对象
- `connect::util`：`shut_close()` 用于隧道结束后的优雅关闭
- [[core/connect/pool/pool]]：连接池提供复用的 TCP 连接
- [[core/context/context]]：`session` 上下文提供缓冲区大小、统计信息和账户租约
- [[core/transport/overview]]：`transmission` 接口定义了 `async_read_some` / `async_write_some`
- [[core/account/entry]]：账户用量累加（`accumulate_uplink` / `accumulate_downlink`）

## 注意事项

1. **任一方向断开即终止**：`operator||` 在任一协程完成时返回，另一方向被取消
2. **缓冲区最小值**：缓冲区大小至少 2 字节，确保每个方向至少 1 字节
3. **日志标签**：使用 `[Connect.Tunnel]` 前缀
4. **流量统计在关闭前执行**：先刷写统计再关闭连接，确保完整性
5. **complete 模式默认**：大多数场景应使用完整写入，避免数据丢失
