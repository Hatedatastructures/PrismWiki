---
title: "Smux 多路复用"
category: "mihomo"
type: ref
module: ref/mihomo
source: "github.com/metacubex/smux"
tags: [mihomo, smux, mux, 多路复用, xtaci]
created: 2026-05-17
updated: 2026-05-17
related: [trojan, kcptun, gost-plugin]
layer: ref
---

# Smux 多路复用

**类别**: Mihomo 多路复用参考

## 概述

Smux 是 xtaci 开发的轻量级 TCP 多路复用协议，由 `github.com/metacubex/smux` 库实现。它设计简单高效，帧头仅 12 字节，适合代理场景的高并发连接复用。

Smux 主要特点：
- 轻量级帧结构（12 bytes header）
- 简单的窗口流控制
- Keepalive 心跳机制
- 高性能设计

## 帧格式

### Smux 帧结构

```
Smux Frame（12 bytes header + data）：
┌────────────────────────────────────────────┐
│ Version: 1 byte (0x01)                     │
│ Command: 1 byte                            │
│ Stream ID: 4 bytes (大端序)                 │
│ Length: 4 bytes (数据长度)                  │
│ Timestamp: 2 bytes                         │
├────────────────────────────────────────────┤
│ Data: variable (Length bytes)              │
└────────────────────────────────────────────┘
```

### Command 类型

| Command | 值 | 说明 |
|---------|-----|------|
| SYN | 0x00 | 同步，创建新流 |
| FIN | 0x01 | 结束，关闭流 |
| PSH | 0x02 | 推送，数据帧 |
| NOP | 0x03 | 空操作 |
| UPD | 0x04 | 窗口更新 |
| KPA | 0x05 | Keepalive |
| BRD | 0x06 | 广播 |

## 配置参数

### Smux 默认配置

```go
// smux.DefaultConfig()
var DefaultConfig = Config{
    MaxReceiveBuffer:   256 * 1024,  // 256KB
    MaxStreamBuffer:    256 * 1024,  // 256KB
    KeepAliveInterval:  10 * time.Second,
    KeepAliveTimeout:   30 * time.Second,
    Version:            1,
}
```

### Kcptun Smux 配置

```yaml
# Kcptun 内置 Smux
plugin-opts:
  smuxver: 1        # Smux 版本
  smuxbuf: 4194304  # Smux 缓冲区（4MB）
  framesize: 8192   # 帧大小
  streambuf: 2097152  # Stream 缓冲区
  keepalive: 10     # Keepalive（秒）
```

## 源码实现

### Mihomo Smux 使用

```go
// 文件: listener/trojan/server.go
// Trojan 服务端 Smux

case trojan.CommandMux:
    smuxConfig := smux.DefaultConfig()
    smuxConfig.KeepAliveDisabled = true
    session, err := smux.Server(conn, smuxConfig)
    if err != nil {
        return
    }
    defer session.Close()
    
    for {
        stream, err := session.AcceptStream()
        if err != nil {
            return
        }
        go l.handleConn(true, stream, tunnel, additions...)
    }
```

```go
// 文件: transport/gost-plugin/websocket.go
// Gost-Plugin Mux

if option.Mux {
    config := smux.DefaultConfig()
    config.KeepAliveDisabled = true
    
    session, err := smux.Client(conn, config)
    if err != nil {
        return nil, err
    }
    
    stream, err := session.OpenStream()
    if err != nil {
        session.Close()
        return nil, err
    }
    
    return &muxConn{
        Conn:    stream,
        session: session,
    }, nil
}
```

### Kcptun Smux

```go
// 文件: transport/kcptun/client.go

smuxConfig := smux.DefaultConfig()
smuxConfig.Version = config.SmuxVer
smuxConfig.MaxReceiveBuffer = config.SmuxBuf
smuxConfig.MaxStreamBuffer = config.StreamBuf
smuxConfig.MaxFrameSize = config.FrameSize
smuxConfig.KeepAliveInterval = time.Duration(config.KeepAlive) * time.Second

if smuxConfig.KeepAliveInterval >= smuxConfig.KeepAliveTimeout {
    smuxConfig.KeepAliveTimeout = 3 * smuxConfig.KeepAliveInterval
}

// Smux 客户端会话
return smux.Client(netConn, smuxConfig)
```

## 与 Prism 兼容性

### Prism Smux 实现

```cpp
// 文件: src/prism/multiplex/smux/session.hpp
namespace psm::multiplex::smux {

struct smux_config {
    uint32_t max_receive_buffer = 256 * 1024;
    uint32_t max_stream_buffer = 256 * 1024;
    std::chrono::seconds keepalive_interval{10};
    std::chrono::seconds keepalive_timeout{30};
    uint8_t version = 1;
    bool keepalive_disabled = false;
};

class smux_session : public mux_session {
public:
    explicit smux_session(net::stream& underlying, const smux_config& cfg);
    
    auto open_stream() -> net::awaitable<mux_stream> override;
    auto accept_stream() -> net::awaitable<mux_stream> override;
    auto close() -> net::awaitable<void> override;
    
private:
    net::stream& underlying_;
    smux_config cfg_;
    std::map<uint32_t, smux_stream> streams_;
    uint32_t next_stream_id_ = 1;
};

class smux_stream : public mux_stream {
public:
    auto read(std::span<std::byte> buffer)
        -> net::awaitable<size_t>;
    auto write(std::span<const std::byte> data)
        -> net::awaitable<size_t>;
    auto close() -> net::awaitable<void>;
    
private:
    uint32_t stream_id_;
    smux_session& session_;
    uint32_t window_ = 0;
};

} // namespace psm::multiplex::smux
```

### 帧构造

```cpp
// 文件: src/prism/multiplex/smux/craft.hpp
auto syn(uint32_t stream_id) -> std::vector<std::byte> {
    std::vector<std::byte> frame(12);
    
    frame[0] = std::byte{0x01};       // Version
    frame[1] = std::byte{0x00};       // Command: SYN
    
    // Stream ID (大端序)
    frame[2] = std::byte{(stream_id >> 24) & 0xFF};
    frame[3] = std::byte{(stream_id >> 16) & 0xFF};
    frame[4] = std::byte{(stream_id >> 8) & 0xFF};
    frame[5] = std::byte{stream_id & 0xFF};
    
    // Length: 0
    // Timestamp: 当前时间
    
    return frame;
}

auto psh(uint32_t stream_id, std::span<const std::byte> data)
    -> std::vector<std::byte> {
    
    std::vector<std::byte> frame(12 + data.size());
    
    frame[0] = std::byte{0x01};
    frame[1] = std::byte{0x02};  // PSH
    
    // Stream ID
    // Length
    // Data
    
    return frame;
}
```

## 工作流程

### 流创建

```
流创建流程：
┌─────────────────────────────────────────────┐
│ Client                    Server             │
│   |                         |               │
│   |-- SYN (Stream 1) ------>|              │
│   |                         |               │
│   |<- SYN ACK (Stream 1) ---|              │
│   |   (服务端自动确认)        |               │
│   |                         |               │
│   |-- SYN (Stream 2) ------>|              │
│   |                         |               │
│   |<- SYN ACK (Stream 2) ---|              │
│   |                         |               │
└─────────────────────────────────────────────┘
```

### 数据传输

```
数据传输：
┌─────────────────────────────────────────────┐
│ Client                    Server             │
│   |                         |               │
│   |-- PSH (Stream 1, data) ->|             │
│   |                         |               │
│   |-- PSH (Stream 2, data) ->|             │
│   |                         |               │
│   |<- PSH (Stream 1, data) --|              │
│   |                         |               │
│   |-- UPD (Stream 1, win) -->|             │
│   |   窗口更新               |              │
│   |                         |               │
└─────────────────────────────────────────────┘
```

## 相关链接

- [[ref/mihomo/mux/overview|Mux 概览]] — 多路复用总览
- [[ref/mihomo/mux/yamux|Yamux]] — Yamux 协议
- [[ref/mihomo/mux/singmux|Sing-Mux]] — Sing-Mux 协议
- [[ref/protocol/smux-yamux|Smux 与 Yamux]] — 协议规范
- [[ref/mihomo/transport/kcptun|Kcptun]] — Kcptun 传输