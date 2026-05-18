---
title: "Yamux 多路复用"
category: "mihomo"
type: ref
module: ref/mihomo
source: "github.com/hashicorp/yamux"
tags: [mihomo, yamux, mux, 多路复用, hashicorp]
created: 2026-05-17
updated: 2026-05-17
related: [trojan, vmess]
layer: ref
---

# Yamux 多路复用

**类别**: Mihomo 多路复用参考

## 概述

Yamux 是 HashiCorp 开发的 TCP 多路复用协议，功能比 Smux 更丰富。它提供完善的窗口流控制、Ping/Pong 心跳机制、GoAway 会话关闭等功能，适合复杂的应用场景。

Yamux 主要特点：
- 功能丰富的帧结构
- 窗口流控制机制
- Ping/Pong 双向心跳
- GoAway 会话关闭
- Flags 组合支持

## 帧格式

### Yamux 帧结构

```
Yamux Frame（12 bytes header + data）：
┌────────────────────────────────────────────┐
│ Version: 1 byte (0x00)                     │
│ Type: 1 byte                               │
│ Flags: 2 bytes                             │
│ Stream ID: 4 bytes (大端序)                 │
│ Length: 4 bytes (数据长度或窗口值)          │
├────────────────────────────────────────────┤
│ Data: variable (Length bytes)              │
└────────────────────────────────────────────┘
```

### Type 类型

| Type | 值 | 说明 |
|------|-----|------|
| Data | 0x00 | 数据帧 |
| Ping | 0x01 | 心跳请求/响应 |
| GoAway | 0x02 | 关闭会话 |

### Flags

| Flag | 值 | 说明 |
|------|-----|------|
| SYN | 0x0001 | 同步，创建流 |
| ACK | 0x0002 | 确认 |
| FIN | 0x0004 | 结束，关闭流 |
| RST | 0x0008 | 重置流 |

Flags 可组合使用：
- `SYN|ACK` = 0x0003：确认创建
- `FIN|ACK` = 0x0006：确认关闭

## 与 Smux 对比

| 特性 | Smux | Yamux |
|------|------|-------|
| 帧头大小 | 12 bytes | 12 bytes |
| 流控制 | 简单窗口 | 窗口机制 |
| 心跳 | KPA 单向 | Ping/Pong |
| 会话关闭 | TCP FIN | GoAway |
| Flags | 无 | SYN/ACK/FIN/RST |
| 复杂度 | 低 | 中 |

## 配置参数

Yamux 通常不作为独立配置，而是通过 Sing-Mux 使用。

### Yamux 默认配置

```go
// Yamux DefaultConfig
var DefaultConfig = Config{
    AcceptBacklog:       256,
    EnableKeepAlive:     true,
    KeepAliveInterval:   30 * time.Second,
    ConnectionWriteTimeout: 10 * time.Second,
    MaxStreamWindowSize: 256 * 1024,
    LogOutput:           os.Stderr,
}
```

## 源码实现

### Yamux 帧类型

```go
// Yamux 帧常量
const (
    // Type
    typeData    uint8 = 0x00
    typePing    uint8 = 0x01
    typeGoAway  uint8 = 0x02
    
    // Flags
    flagSYN uint16 = 0x0001
    flagACK uint16 = 0x0002
    flagFIN uint16 = 0x0004
    flagRST uint16 = 0x0008
)

// Yamux 帧结构
type header struct {
    Version     uint8
    Type        uint8
    Flags       uint16
    StreamID    uint32
    Length      uint32
}
```

### Yamux 会话

```go
// Yamux Session
type Session struct {
    config       *Config
    conn         net.Conn
    streams      map[uint32]*Stream
    nextStreamID uint32
    shutdown     bool
}

func (s *Session) OpenStream() (*Stream, error) {
    streamID := s.nextStreamID
    s.nextStreamID += 2  // 客户端使用奇数 ID
    
    // 发送 Data(SYN) 帧
    hdr := header{
        Version:  0x00,
        Type:     typeData,
        Flags:    flagSYN,
        StreamID: streamID,
        Length:   s.config.MaxStreamWindowSize,
    }
    s.sendHeader(hdr)
    
    // 创建 Stream
    stream := &Stream{
        id:     streamID,
        session: s,
        recvWindow: s.config.MaxStreamWindowSize,
    }
    s.streams[streamID] = stream
    
    return stream, nil
}

func (s *Session) Ping() error {
    hdr := header{
        Version:  0x00,
        Type:     typePing,
        Flags:    flagSYN,
        Length:   uint32(time.Now().UnixNano()),
    }
    return s.sendHeader(hdr)
}
```

## 与 Prism 兼容性

### Prism Yamux 实现

```cpp
// 文件: src/prism/multiplex/yamux/session.hpp
namespace psm::multiplex::yamux {

enum class frame_type : uint8_t {
    data   = 0x00,
    ping   = 0x01,
    goaway = 0x02
};

enum class frame_flag : uint16_t {
    syn = 0x0001,
    ack = 0x0002,
    fin = 0x0004,
    rst = 0x0008
};

struct yamux_config {
    uint32_t accept_backlog = 256;
    bool enable_keepalive = true;
    std::chrono::seconds keepalive_interval{30};
    std::chrono::seconds connection_write_timeout{10};
    uint32_t max_stream_window_size = 256 * 1024;
};

class yamux_session : public mux_session {
public:
    auto open_stream() -> net::awaitable<mux_stream> override;
    auto accept_stream() -> net::awaitable<mux_stream> override;
    auto ping() -> net::awaitable<void>;
    auto goaway(uint32_t error_code) -> net::awaitable<void>;
    auto close() -> net::awaitable<void> override;
    
private:
    net::stream& underlying_;
    yamux_config cfg_;
    std::map<uint32_t, yamux_stream> streams_;
    uint32_t next_stream_id_ = 1;  // 客户端使用奇数 ID
};

} // namespace psm::multiplex::yamux
```

### Yamux 帧构造

```cpp
// 文件: src/prism/multiplex/yamux/craft.hpp
auto data_syn(uint32_t stream_id, uint32_t window_size)
    -> std::vector<std::byte> {
    
    std::vector<std::byte> frame(12);
    
    frame[0] = std::byte{0x00};       // Version
    frame[1] = std::byte{0x00};       // Type: Data
    
    // Flags: SYN
    frame[2] = std::byte{0x00};
    frame[3] = std::byte{0x01};
    
    // Stream ID
    append_u32_be(frame, stream_id, 4);
    
    // Length: 窗口大小
    append_u32_be(frame, window_size, 8);
    
    return frame;
}

auto ping_syn(uint32_t value)
    -> std::vector<std::byte> {
    
    std::vector<std::byte> frame(12);
    
    frame[0] = std::byte{0x00};
    frame[1] = std::byte{0x01};  // Type: Ping
    frame[2] = std::byte{0x00};
    frame[3] = std::byte{0x01};  // Flags: SYN
    
    // Length: Ping 值
    append_u32_be(frame, value, 8);
    
    return frame;
}
```

## 工作流程

### 流创建

```
Yamux 流创建：
┌─────────────────────────────────────────────┐
│ Client                    Server             │
│   |                         |               │
│   |-- Data(SYN, Stream 1) -->|             │
│   |   Length: 窗口大小        |              │
│   |                         |               │
│   |<- Data(SYN|ACK, Stream 1)-|            │
│   |   确认创建 + 窗口          |               │
│   |                         |               │
│   |-- Data(SYN, Stream 2) -->|             │
│   |                         |               │
│   |<- Data(SYN|ACK, Stream 2)-|            │
│   |                         |               │
└─────────────────────────────────────────────┘
```

### Ping/Pong 心跳

```
Ping/Pong：
┌─────────────────────────────────────────────┐
│ Client                    Server             │
│   |                         |               │
│   |-- Ping(SYN) ----------->|              │
│   |   Length: 时间戳         |               │
│   |                         |               │
│   |<- Ping(ACK) ------------|              │
│   |   Length: 同样的时间戳   |               │
│   |                         |               │
└─────────────────────────────────────────────┘
```

### GoAway 关闭

```
GoAway 会话关闭：
┌─────────────────────────────────────────────┐
│ Server                    Client             │
│   |                         |               │
│   |-- GoAway -------------->|              │
│   |   Length: 错误码         |               │
│   |                         |               │
│                                             │
│ 错误码：                                     │
│   0x00000000: 正常关闭                      │
│   0x00000001: 协议错误                      │
│   0x00000002: 内部错误                      │
└─────────────────────────────────────────────┘
```

## 相关链接

- [[ref/mihomo/mux/overview|Mux 概览]] — 多路复用总览
- [[ref/mihomo/mux/smux|Smux]] — Smux 协议
- [[ref/mihomo/mux/singmux|Sing-Mux]] — Sing-Mux 协议
- [[ref/protocol/smux-yamux|Smux 与 Yamux]] — 协议规范