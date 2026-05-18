---
title: "Smux 与 Yamux 多路复用协议"
category: "protocol"
type: ref
module: ref
source: "https://github.com/xtaci/smux, https://github.com/hashicorp/yamux"
tags: [协议, smux, yamux, 多路复用, 流, tcp, mux]
created: 2026-05-15
updated: 2026-05-17
related: [tcp, trojan, vless, shadowsocks]
layer: ref
---

# Smux 与 Yamux 多路复用协议

**类别**: 协议

## 概述

Smux 和 Yamux 是两种广泛应用于代理领域的 TCP 多路复用协议。多路复用（Multiplexing）允许在单个 TCP 连接上运行多个独立的流（Stream），每个流可以独立传输数据。这种技术可以减少 TCP 连接数量，降低连接建立开销，提高资源利用效率。

在代理场景中，多路复用协议的应用价值：

- **连接复用**：减少 TCP 连接数量，降低开销
- **降低延迟**：避免重复的 TCP/TLS 握手
- **资源效率**：减少服务器连接数压力
- **穿透能力**：在受限网络环境下保持连接

在 Prism 项目中，Smux 和 Yamux 是 Multiplex 模块支持的核心多路复用协议，与 Trojan、VLESS、Shadowsocks 等代理协议配合使用。

### 多路复用的必要性

多路复用解决的问题：

```
传统 TCP 代理问题：
┌────────────────────────────────────────────┐
│ 1. 连接开销                                  │
│    - 每个请求需要建立 TCP 连接                │
│    - TCP 握手：1 RTT                          │
│    - TLS 握手：额外 1-2 RTT                   │
│                                            │
│ 2. 连接数量                                  │
│    - 大量并发请求需要大量连接                 │
│    - 服务器连接数压力大                       │
│                                            │
│ 3. 资源消耗                                  │
│    - 每个连接消耗内存和 CPU                   │
│    - NAT 表项增加                            │
│                                            │
│ 4. 网络环境                                  │
│    - 连接数限制                              │
│    - 连接超时断开                            │
└────────────────────────────────────────────┘

多路复用解决方案：
┌────────────────────────────────────────────┐
│ 1. 单连接多流                                │
│    - 一个 TCP 连接承载多个逻辑流             │
│                                            │
│ 2. 降低开销                                  │
│    - 共享 TCP/TLS 握手                       │
│    - 流创建无额外开销                        │
│                                            │
│ 3. 资源效率                                  │
│    - 减少连接数                              │
│    - 降低内存消耗                            │
│                                            │
│ 4. 连接稳定                                  │
│    - 长连接不易断开                          │
│    - 自动重连机制                            │
└────────────────────────────────────────────┘
```

### Smux vs Yamux

两种多路复用协议的对比：

| 特性 | Smux | Yamux |
|------|------|-------|
| 来源 | xtaci (2017) | HashiCorp (2014) |
| 帧头大小 | 12 bytes | 12 bytes |
| 流控制 | 简单窗口 | 窗口机制 |
| 心跳机制 | Keepalive | Ping/Pong |
| 初始窗口 | 固定 | 可配置 |
| 复杂度 | 较低 | 较高 |
| 适用场景 | 简单代理 | 复杂应用 |
| 库支持 | Go (官方) | Go (官方) |

### 多路复用原理

多路复用的基本原理：

```
多路复用原理：
┌────────────────────────────────────────────┐
│                                            │
│  Client                    Server           │
│    |                         |              │
│    |--- TCP/TLS 连接 -------|              │
│    |   (一条物理连接)         |              │
│    |                         |              │
│    |========= 多路复用帧 ==========|        │
│    |                         |              │
│    |  帧1: Stream ID=1       |              │
│    |  帧2: Stream ID=2       |              │
│    |  帧3: Stream ID=1       |              │
│    |  帧4: Stream ID=3       |              │
│    |  ...                    |              │
│    |                         |              │
│  流1: 代理请求A              │
│  流2: 代理请求B              │
│  流3: 代理请求C              │
│                                            │
└────────────────────────────────────────────┘

关键概念：
┌────────────────────────────────────────────┐
│ Stream ID: 标识不同的逻辑流                 │
│ Frame: 数据传输的基本单位                   │
│ Window: 流控制机制                          │
│ Session: 整个多路复用连接                   │
└────────────────────────────────────────────┘
```

## 协议详解

### Smux 协议

Smux 是轻量级多路复用协议，由 xtaci 开发。

#### Smux 帧格式

Smux 帧的完整结构：

```
Smux Frame (12 bytes header + data):
┌────────────────────────────────────────────┐
│ Version: 1 byte (0x01)                     │
│ Command: 1 byte                            │
│ Stream ID: 4 bytes (大端序)                 │
│ Length: 4 bytes (数据长度)                  │
│ Timestamp: 2 bytes (可选，用于 keepalive)   │
├────────────────────────────────────────────┤
│ Data: variable (Length bytes)              │
└────────────────────────────────────────────┘

Version:
┌────────────────────────────────────────────┐
│ 0x01: 当前版本                              │
└────────────────────────────────────────────┘

Command 类型：
┌────────────────────────────────────────────┐
│ 0x00: SYN (同步，创建新流)                  │
│ 0x01: FIN (结束，关闭流)                    │
│ 0x02: PSH (推送，数据帧)                    │
│ 0x03: NOP (空操作)                          │
│ 0x04: UPD (窗口更新)                        │
│ 0x05: KPA (Keepalive)                       │
│ 0x06: BRD (广播)                            │
└────────────────────────────────────────────┘
```

#### Smux 帧类型详解

```
SYN 帧 (创建流)：
┌────────────────────────────────────────────┐
│ Version: 0x01                              │
│ Command: 0x00 (SYN)                        │
│ Stream ID: 新流的 ID                        │
│ Length: 0 (通常无数据)                      │
│ Timestamp: 当前时间                         │
│                                            │
│ 作用：                                      │
│   - 客户端创建新流                          │
│   - 服务器确认流创建                        │
└────────────────────────────────────────────┘

FIN 帧 (关闭流)：
┌────────────────────────────────────────────┐
│ Version: 0x01                              │
│ Command: 0x01 (FIN)                        │
│ Stream ID: 要关闭的流 ID                    │
│ Length: 0                                  │
│ Timestamp: -                               │
│                                            │
│ 作用：                                      │
│   - 单向关闭流                              │
│   - 类似 TCP FIN                            │
└────────────────────────────────────────────┘

PSH 帧 (数据推送)：
┌────────────────────────────────────────────┐
│ Version: 0x01                              │
│ Command: 0x02 (PSH)                        │
│ Stream ID: 目标流 ID                        │
│ Length: 数据长度                            │
│ Timestamp: -                               │
│ Data: 实际数据                              │
│                                            │
│ 作用：                                      │
│   - 传输数据                                │
│   - 最常用的帧类型                          │
└────────────────────────────────────────────┘

UPD 帧 (窗口更新)：
┌────────────────────────────────────────────┐
│ Version: 0x01                              │
│ Command: 0x04 (UPD)                        │
│ Stream ID: 目标流 ID                        │
│ Length: 窗口增量                            │
│ Timestamp: -                               │
│                                            │
│ 作用：                                      │
│   - 更新接收窗口                            │
│   - 流控制机制                              │
└────────────────────────────────────────────┘

KPA 帧 (Keepalive)：
┌────────────────────────────────────────────┐
│ Version: 0x01                              │
│ Command: 0x05 (KPA)                        │
│ Stream ID: 0                               │
│ Length: 0                                  │
│ Timestamp: 当前时间                         │
│                                            │
│ 作用：                                      │
│   - 保持连接活跃                            │
│   - 检测连接状态                            │
└────────────────────────────────────────────┘
```

#### Smux 流程

Smux 的完整工作流程：

```
Smux 工作流程：

Client                    Server
  |                           |
  |--- TCP 连接 -------------->|
  |                           |
  |--- SYN (Stream 1) -------->|  创建流1
  |                           |
  |<-- SYN ACK (Stream 1) -----|  确认流1
  |                           |
  |--- SYN (Stream 2) -------->|  创建流2
  |                           |
  |<-- SYN ACK (Stream 2) -----|  确认流2
  |                           |
  |--- PSH (Stream 1, data) -->|  流1数据
  |                           |
  |--- PSH (Stream 2, data) -->|  流2数据
  |                           |
  |<-- PSH (Stream 1, data) ---|  流1响应
  |                           |
  |<-- PSH (Stream 2, data) ---|  流2响应
  |                           |
  |--- FIN (Stream 1) -------->|  关闭流1
  |                           |
  |<-- FIN (Stream 1) ---------|
  |                           |
  |--- KPA ------------------>|  Keepalive
  |<-- KPA -------------------|
  |                           |
```

#### Smux 流控制

Smux 的简单流控制：

```
Smux 流控制：
┌────────────────────────────────────────────┐
│ 窗口机制：                                   │
│   - 每个流有接收窗口                         │
│   - 发送方不能超过窗口                       │
│   - 接收方发送 UPD 更新窗口                 │
│                                            │
│ 窗口大小：                                   │
│   - 默认: 配置指定                          │
│   - 动态调整: 通过 UPD 帧                   │
│                                            │
│ 工作原理：                                   │
│   1. 发送方检查窗口                         │
│   2. 发送数据后减少窗口                     │
│   3. 接收方处理后发送 UPD                   │
│   4. 发送方收到 UPD 增加窗口                │
└────────────────────────────────────────────┘
```

#### Smux 配置参数

```
Smux 配置：
┌────────────────────────────────────────────┐
│ max_streams: 最大流数量                      │
│   - 推荐: 64-256                            │
│                                            │
│ buffer_size: 每个流的缓冲区大小              │
│   - 推荐: 32KB-256KB                        │
│                                            │
│ keepalive_interval: Keepalive 间隔          │
│   - 推荐: 10-30 秒                          │
│                                            │
│ version: 协议版本                           │
│   - 当前: 1                                 │
└────────────────────────────────────────────┘
```

### Yamux 协议

Yamux 是 HashiCorp 开发的多路复用协议，功能更丰富。

#### Yamux 帧格式

Yamux 帧的完整结构：

```
Yamux Frame (12 bytes header + data):
┌────────────────────────────────────────────┐
│ Version: 1 byte (0x00)                     │
│ Type: 1 byte                               │
│ Flags: 2 bytes                             │
│ Stream ID: 4 bytes (大端序)                 │
│ Length: 4 bytes (数据长度或窗口值)          │
├────────────────────────────────────────────┤
│ Data: variable (Length bytes)              │
└────────────────────────────────────────────┘

Version:
┌────────────────────────────────────────────┐
│ 0x00: 当前版本                              │
└────────────────────────────────────────────┘

Type 类型：
┌────────────────────────────────────────────┐
│ 0x00: Data (数据帧)                         │
│ 0x01: Ping (心跳请求)                       │
│ 0x02: GoAway (关闭会话)                     │
└────────────────────────────────────────────┘

Flags:
┌────────────────────────────────────────────┐
│ 0x0001: SYN (同步，创建流)                  │
│ 0x0002: ACK (确认)                          │
│ 0x0004: FIN (结束，关闭流)                  │
│ 0x0008: RST (重置流)                        │
│                                            │
│ 可组合使用：                                 │
│   SYN|ACK = 0x0003: 确认创建                │
│   FIN|ACK = 0x0005: 确认关闭                │
└────────────────────────────────────────────┘
```

#### Yamux 帧类型详解

```
Data 帧 (携带数据或窗口更新)：
┌────────────────────────────────────────────┐
│ Version: 0x00                              │
│ Type: 0x00 (Data)                          │
│ Flags:                                     │
│   SYN: 创建流                               │
│   FIN: 关闭流                               │
│   ACK: 确认                                 │
│   无 Flags: 纯数据                          │
│ Stream ID: 目标流 ID                        │
│ Length: 数据长度                            │
│ Data: 数据内容                              │
└────────────────────────────────────────────┘

Ping 帧 (心跳)：
┌────────────────────────────────────────────┐
│ Version: 0x00                              │
│ Type: 0x01 (Ping)                          │
│ Flags:                                     │
│   SYN: Ping 请求                            │
│   ACK: Ping 响应                            │
│ Stream ID: 0                               │
│ Length: Ping 值                             │
│                                            │
│ Ping/Pong 机制：                            │
│   客户端发送 Ping(SYN)                      │
│   服务器回复 Ping(ACK)                      │
│   用于检测连接状态                          │
└────────────────────────────────────┘

GoAway 帧 (关闭会话)：
┌────────────────────────────────────────────┐
│ Version: 0x00                              │
│ Type: 0x02 (GoAway)                        │
│ Flags: 0x0000                              │
│ Stream ID: 0                               │
│ Length: 错误码                              │
│                                            │
│ 错误码：                                     │
│   0x00000000: 正常关闭                      │
│   0x00000001: 协议错误                      │
│   0x00000002: 内部错误                      │
└────────────────────────────────────────────┘
```

#### Yamux 流程

Yamux 的完整工作流程：

```
Yamux 工作流程：

Client                    Server
  |                           |
  |--- TCP 连接 -------------->|
  |                           |
  |--- Data(SYN, Stream 1) --->|  创建流1
  |                           |
  |<-- Data(SYN|ACK, Stream 1)-|  确认流1
  |                           |
  |--- Data(SYN, Stream 2) --->|  创建流2
  |                           |
  |<-- Data(SYN|ACK, Stream 2)-|  确认流2
  |                           |
  |--- Data(Stream 1, data) -->|  流1数据
  |                           |
  |--- Data(Stream 2, data) -->|  流2数据
  |                           |
  |<-- Data(Stream 1, data) ---|  流1响应
  |                           |
  |--- Ping(SYN) ------------->|  Ping
  |<-- Ping(ACK) --------------|  Pong
  |                           |
  |--- Data(FIN, Stream 1) --->|  关闭流1
  |                           |
  |<-- Data(FIN|ACK, Stream 1)-|  确认关闭
  |                           |
```

#### Yamux 流控制

Yamux 的窗口流控制：

```
Yamux 流控制：
┌────────────────────────────────────────────┐
│ 窗口机制：                                   │
│   - 初始窗口: 配置指定                       │
│   - 每个流独立窗口                           │
│   - 通过 Data 帧更新                        │
│                                            │
│ 窗口增量：                                   │
│   - Data(SYN): 设置初始窗口                 │
│   - Data(无Flags): 数据消耗窗口             │
│   - Data(ACK): 窗口增量                     │
│                                            │
│ 工作原理：                                   │
│   1. SYN 时设置初始窗口                     │
│   2. 发送数据减少窗口                       │
│   3. 接收方处理后发送 ACK                   │
│   4. Length 字段为窗口增量                  │
│                                            │
│ 窗口限制：                                   │
│   - 发送方等待窗口                           │
│   - 窗口不足时阻塞                          │
│   - 避免接收方溢出                          │
└────────────────────────────────────────────┘
```

#### Yamux 配置参数

```
Yamux 配置：
┌────────────────────────────────────────────┐
│ max_streams: 最大流数量                      │
│   - 推荐: 64-256                            │
│                                            │
│ buffer_size: 每个流的缓冲区大小              │
│   - 推荐: 32KB-256KB                        │
│                                            │
│ initial_window: 初始窗口大小                 │
│   - 推荐: 256KB-1MB                         │
│                                            │
│ enable_ping: 是否启用 Ping/Pong             │
│   - 推荐: true                              │
│                                            │
│ ping_interval: Ping 间隔                    │
│   - 推荐: 15-30 秒                          │
│                                            │
│ ping_timeout: Ping 超时                     │
│   - 推荐: 30-60 秒                          │
└────────────────────────────────────────────┘
```

### Smux vs Yamux 详细对比

```
详细对比：
┌────────────────────────────────────────────┐
│ 帧结构：                                     │
│   Smux: 版本 + 命令 + ID + 长度 + 时间戳     │
│   Yamux: 版本 + 类型 + Flags + ID + 长度    │
│                                            │
│ 流创建：                                     │
│   Smux: SYN 帧                              │
│   Yamux: Data(SYN) 帧                       │
│                                            │
│ 流关闭：                                     │
│   Smux: FIN 帧                              │
│   Yamux: Data(FIN) 帧                       │
│                                            │
│ 数据传输：                                   │
│   Smux: PSH 帧                              │
│   Yamux: Data 帧                            │
│                                            │
│ 窗口更新：                                   │
│   Smux: UPD 帧                              │
│   Yamux: Data(ACK) 帧                       │
│                                            │
│ 心跳：                                       │
│   Smux: KPA 帧                              │
│   Yamux: Ping/Pong 帧                       │
│                                            │
│ 会话关闭：                                   │
│   Smux: TCP FIN                             │
│   Yamux: GoAway 帧                          │
│                                            │
│ Flags 支持：                                 │
│   Smux: 无                                  │
│   Yamux: SYN/ACK/FIN/RST                    │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### Smux 帧构造

Prism 构造 Smux 帧：

```cpp
// 文件: src/prism/multiplex/smux/craft.hpp
namespace psm::multiplex::smux::craft {

auto syn(uint32_t stream_id)
    -> std::vector<std::byte>
{
    std::vector<std::byte> frame(12);
    
    frame[0] = std::byte{0x01};       // Version
    frame[1] = std::byte{0x00};       // Command: SYN
    
    // Stream ID (大端序)
    frame[2] = std::byte{(stream_id >> 24) & 0xFF};
    frame[3] = std::byte{(stream_id >> 16) & 0xFF};
    frame[4] = std::byte{(stream_id >> 8) & 0xFF};
    frame[5] = std::byte{stream_id & 0xFF};
    
    // Length: 0
    frame[6] = frame[7] = frame[8] = frame[9] = std::byte{0};
    
    // Timestamp
    auto timestamp = current_timestamp();
    frame[10] = std::byte{(timestamp >> 8) & 0xFF};
    frame[11] = std::byte{timestamp & 0xFF};
    
    return frame;
}

auto psh(uint32_t stream_id,
         std::span<const std::byte> data)
    -> std::vector<std::byte>
{
    std::vector<std::byte> frame(12 + data.size());
    
    frame[0] = std::byte{0x01};       // Version
    frame[1] = std::byte{0x02};       // Command: PSH
    
    // Stream ID
    append_u32_be(frame, stream_id, 2);
    
    // Length
    append_u32_be(frame, data.size(), 6);
    
    // Timestamp (可选)
    frame[10] = frame[11] = std::byte{0};
    
    // Data
    std::copy(data.begin(), data.end(), frame.begin() + 12);
    
    return frame;
}

auto fin(uint32_t stream_id)
    -> std::vector<std::byte>
{
    // ... 类似 SYN
}

} // namespace psm::multiplex::smux::craft
```

### Yamux 帧构造

Prism 构造 Yamux 帧：

```cpp
// 文件: src/prism/multiplex/yamux/craft.hpp
namespace psm::multiplex::yamux::craft {

auto data_syn(uint32_t stream_id)
    -> std::vector<std::byte>
{
    std::vector<std::byte> frame(12);
    
    frame[0] = std::byte{0x00};       // Version
    frame[1] = std::byte{0x00};       // Type: Data
    
    // Flags: SYN
    frame[2] = std::byte{0x00};
    frame[3] = std::byte{0x01};
    
    // Stream ID
    append_u32_be(frame, stream_id, 4);
    
    // Length: 窗口大小
    append_u32_be(frame, initial_window, 8);
    
    return frame;
}

auto data(uint32_t stream_id,
          std::span<const std::byte> data)
    -> std::vector<std::byte>
{
    std::vector<std::byte> frame(12 + data.size());
    
    frame[0] = std::byte{0x00};       // Version
    frame[1] = std::byte{0x00};       // Type: Data
    
    // Flags: 0 (纯数据)
    frame[2] = frame[3] = std::byte{0};
    
    // Stream ID
    append_u32_be(frame, stream_id, 4);
    
    // Length
    append_u32_be(frame, data.size(), 8);
    
    // Data
    std::copy(data.begin(), data.end(), frame.begin() + 12);
    
    return frame;
}

auto ping(uint32_t value)
    -> std::vector<std::byte>
{
    // Ping(SYN) 帧
}

} // namespace psm::multiplex::yamux::craft
```

### 帧解析

Prism 解析多路复用帧：

```cpp
// 文件: src/prism/multiplex/smux/frame.hpp
namespace psm::multiplex::smux::frame {

auto parse(std::span<const std::byte> data)
    -> outcome::result<mux_frame>
{
    if (data.size() < 12) {
        return fault::code::insufficient_data;
    }
    
    mux_frame frame;
    
    frame.version = static_cast<uint8_t>(data[0]);
    if (frame.version != 0x01) {
        return fault::code::invalid_version;
    }
    
    frame.command = static_cast<uint8_t>(data[1]);
    frame.stream_id = read_u32_be(data, 2);
    frame.length = read_u32_be(data, 6);
    
    if (data.size() < 12 + frame.length) {
        return fault::code::insufficient_data;
    }
    
    frame.data = data.subspan(12, frame.length);
    
    return frame;
}

} // namespace psm::multiplex::smux::frame
```

### Session 管理

Prism 的多路复用会话：

```cpp
// 文件: src/prism/multiplex/session.hpp
class mux_session {
public:
    auto create_stream()
        -> net::awaitable<mux_stream>
    {
        auto stream_id = next_stream_id_++;
        
        // 发送 SYN 帧
        auto syn_frame = craft::syn(stream_id);
        co_await net::async_write(socket_,
            net::buffer(syn_frame),
            net::use_awaitable);
        
        // 等待确认
        // ...
        
        // 创建流对象
        auto stream = mux_stream(stream_id, this);
        streams_[stream_id] = stream;
        
        co_return stream;
    }
    
    auto close_stream(uint32_t stream_id)
        -> net::awaitable<void>
    {
        auto fin_frame = craft::fin(stream_id);
        co_await net::async_write(socket_,
            net::buffer(fin_frame),
            net::use_awaitable);
        
        streams_.erase(stream_id);
    }
    
    auto process_frame(const mux_frame& frame)
        -> net::awaitable<void>
    {
        switch (frame.command) {
            case 0x00: // SYN
                handle_syn(frame);
                break;
            case 0x01: // FIN
                handle_fin(frame);
                break;
            case 0x02: // PSH
                handle_data(frame);
                break;
            case 0x05: // KPA
                handle_keepalive(frame);
                break;
        }
    }
    
private:
    net::tcp::socket socket_;
    std::map<uint32_t, mux_stream> streams_;
    uint32_t next_stream_id_ = 1;
};
```

## 最佳实践

### 配置选择

1. **流数量**：根据负载设置合理上限（64-256）
2. **缓冲区大小**：平衡内存和吞吐量
3. **Keepalive**：设置合理的间隔（10-30秒）
4. **窗口大小**：根据网络延迟调整

### 性能优化

1. **连接复用**：保持长连接
2. **流管理**：及时关闭空闲流
3. **缓冲策略**：合理设置缓冲区
4. **并行处理**：多流并行处理

### 稳定性保障

1. **心跳机制**：保持连接活跃
2. **错误处理**：正确处理异常帧
3. **超时机制**：设置合理的超时
4. **资源清理**：清理失效的流

### 与代理协议配合

1. **Trojan + Smux/Yamux**：常见组合
2. **VLESS + Smux/Yamux**：支持多路复用
3. **Shadowsocks + Smux**：可选增强
4. **协议选择**：根据场景选择合适的组合

## 常见问题

### Q1: Smux 和 Yamux 哪个更好？

选择建议：

- **Smux**：简单、轻量、适合大多数代理场景
- **Yamux**：功能丰富、适合复杂应用

一般代理场景推荐 Smux。

### Q2: 多路复用会增加延迟吗？

延迟影响：

- 首次连接：略高（建立 Session）
- 后续请求：更低（无需新连接）
- 整体吞吐：更好（连接复用）

### Q3: 最大流数量如何设置？

设置建议：

- 根据并发需求设置
- 太高：内存压力大
- 太低：并发受限
- 推荐：64-256

### Q4: 如何处理流阻塞？

阻塞处理：

- 窗口控制：等待窗口更新
- 流分离：不同流独立处理
- 超时机制：设置流超时
- 优先级：可设置流优先级

### Q5: Keepalive 和 Ping 有什么区别？

区别：

- **Smux KPA**：单向心跳
- **Yamux Ping/Pong**：双向确认

两者都用于保持连接活跃。

### Q6: Prism 如何选择多路复用协议？

Prism 的选择：

- 配置指定协议类型
- 根据目标服务器支持
- Smux 默认用于简单场景
- Yamux 用于需要更多功能的场景

### Q7: 多路复用对 NAT 有什么影响？

NAT 影响：

- 减少连接数：减轻 NAT 表压力
- 长连接：不易被 NAT 超时
- 但可能：流量更集中

### Q8: 如何调试多路复用问题？

调试方法：

- 检查帧格式
- 分析流状态
- 监控窗口大小
- 日志记录关键事件

## 参考资料

- [Smux - Simple Multiplexer](https://github.com/xtaci/smux)
- [Yamux - Yet Another Multiplexer](https://github.com/hashicorp/yamux)
- [Multiplexing Protocol Design](https://www.haproxy.com/blog/multiplexing-protocols-explained/)

## 相关知识

- [[ref/network/tcp|TCP]] — TCP 协议基础
- [[ref/protocol/trojan|Trojan 协议]] — Trojan 代理
- [[ref/protocol/vless|VLESS 协议]] — VLESS 代理
- [[ref/protocol/shadowsocks|Shadowsocks 协议]] — SS2022 协议