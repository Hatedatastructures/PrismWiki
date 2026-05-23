---
title: tunnel
layer: core
source: I:/code/Prism/include/prism/connect/tunnel/tunnel.hpp
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
auto tunnel(shared_transmission inbound,
            shared_transmission outbound,
            const context::session &ctx,
            bool complete_write = true)
    -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `inbound` | `shared_transmission` | 入站传输对象（客户端连接） |
| `outbound` | `shared_transmission` | 出站传输对象（上游连接） |
| `ctx` | `const context::session &` | 会话上下文，提供缓冲区配置和统计信息 |
| `complete_write` | `bool` | 是否使用完整写入语义（默认 `true`） |

### 返回值

`net::awaitable<void>`：隧道结束后完成（任一方向断开即终止）

### complete_write 参数

| 值 | 行为 | 适用场景 |
|----|------|----------|
| `true`（默认） | 使用 `transport::async_write()` 循环写入直到所有数据发送完毕 | 大多数协议转发 |
| `false` | 使用 `async_write_some()` 单次写入，可能部分写入 | 特殊场景（如流控） |

## 内部实现

### 缓冲区分配

```cpp
auto *mr = memory::system::thread_local_pool();
const auto array_size = (std::max)(ctx.buffer_size, 2U);
memory::vector<std::byte> buffer(array_size, mr ? mr : memory::current_resource());
const auto half = buffer.size() / 2;
const auto left = std::span(buffer).first(half);   // 入站 -> 出站 缓冲区
const auto right = std::span(buffer).last(half);    // 出站 -> 入站 缓冲区
```

**关键设计：**
- 单一分配，双缓冲区：从 `thread_local_pool()` 分配一块连续内存，前半给上传方向，后半给下载方向
- 缓冲区大小至少 2 字节（`max(buffer_size, 2)`）
- 使用 PMR 分配器，避免堆分配

### 方向标识

```cpp
struct forward_context
{
    const trans &from;
    const trans &to;
    const std::span<std::byte> scratch;
    const std::size_t idx;  // 0 = upload, 1 = download
};
```

| idx | 方向 | from | to | 含义 |
|-----|------|------|----|------|
| 0 | upload | inbound | outbound | 客户端 -> 上游 |
| 1 | download | outbound | inbound | 上游 -> 客户端 |

### 数据转发循环

```cpp
auto forward_data = [complete_write, &total_bytes](forward_context context)
    -> net::awaitable<void>
{
    std::error_code ec;
    while (true)
    {
        // 1. 从源端读取数据
        const auto transferred = co_await context.from->async_read_some(context.scratch, ec);
        if (ec || transferred == 0)
            co_return;

        total_bytes[context.idx] += transferred;

        // 2. 写入目标端
        const auto data = context.scratch.first(transferred);
        if (complete_write)
            written = co_await transport::async_write(*context.to, data, ec);
        else
            written = co_await context.to->async_write_some(data, ec);

        if (ec || (complete_write && written < transferred))
            co_return;
    }
};
```

### 并发执行

```cpp
using namespace boost::asio::experimental::awaitable_operators;
co_await (forward_data({inbound, outbound, left, 0}) ||
          forward_data({outbound, inbound, right, 1}));
```

使用 `operator||`（并发等待）同时启动两个方向的转发协程。任一方向完成（读返回 0 或错误），`operator||` 立即返回，另一方向的协程被自动取消。

## 流量统计

### 统计刷写

```cpp
if (ctx.worker_ctx.traffic)
{
    ctx.worker_ctx.traffic->flush_traffic(
        ctx.detected_protocol, total_bytes[0], total_bytes[1]);
}
```

隧道结束后，将上传/下载字节数刷写到 worker 级别的流量统计模块。

**统计字段：**
- `total_bytes[0]`：上传字节数（inbound -> outbound）
- `total_bytes[1]`：下载字节数（outbound -> inbound）
- `ctx.detected_protocol`：识别出的协议类型

### 账户用量累加

```cpp
if (ctx.account_lease)
{
    if (total_bytes[0] > 0)
        account::accumulate_uplink(ctx.account_lease.get(), total_bytes[0]);
    if (total_bytes[1] > 0)
        account::accumulate_downlink(ctx.account_lease.get(), total_bytes[1]);
}
```

如果会话关联了账户租约，将流量累加到账户的上行/下行用量中。这支持按用户的流量配额管理。

### 日志输出

```cpp
trace::info("{} [{}] Transfer: Upload {} B, Download {} B, duration: {} ms",
            TunnelStr, ctx.session_id, up, down, duration_ms);
```

隧道结束后输出传输统计日志，包含会话 ID、上传/下载字节数和持续时间。

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
- [[core/connect/util]]：`shut_close()` 用于隧道结束后的优雅关闭
- [[core/connect/pool/pool]]：连接池提供复用的 TCP 连接
- [[core/context/context]]：`session` 上下文提供缓冲区大小、统计信息和账户租约
- [[core/transport]]：`transmission` 接口定义了 `async_read_some` / `async_write_some`
- [[core/account/entry]]：账户用量累加（`accumulate_uplink` / `accumulate_downlink`）

## 注意事项

1. **任一方向断开即终止**：`operator||` 在任一协程完成时返回，另一方向的协程被取消。这意味着如果上游关闭连接，客户端连接也会被关闭。
2. **缓冲区最小值**：缓冲区大小至少为 2 字节（`max(buffer_size, 2)`），确保每个方向至少 1 字节。
3. **日志标签**：使用 `[Connect.Tunnel]` 前缀，与 [[core/connect/tunnel/forward|forward]] 的 `[Connect.Forward]` 区分。
4. **统计仅在传输量 > 0 时输出**：`if (up > 0 || down > 0)` 条件避免空隧道产生无意义的日志。
5. **流量统计在关闭前执行**：先刷写统计再关闭连接，确保统计数据的完整性。
6. **complete_write 默认为 true**：大多数场景应使用完整写入，避免数据丢失。部分写入可能导致数据截断。
