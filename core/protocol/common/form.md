---
layer: core
source: "include/prism/protocol/common/form.hpp"
title: 传输形态枚举
tags: [protocol, common, form, stream, datagram, tcp, udp]
---

# 传输形态枚举

定义传输层的数据传输形态，用于区分 TCP 流式传输和 UDP 数据报传输。

## 源码位置

`include/prism/protocol/common/form.hpp`

## 枚举定义

```cpp
namespace psm::protocol
{
    enum class form : std::uint8_t
    {
        stream,      // TCP 可靠流传输
        datagram     // UDP 数据报传输
    };
}
```

## 值说明

| 值 | 说明 | 对应协议命令 |
|---|---|---|
| `stream` | 可靠有序字节流 | CONNECT (SOCKS5/Trojan), TCP (VLESS), MUX |
| `datagram` | 不可靠数据报 | UDP_ASSOCIATE (SOCKS5/Trojan), UDP (VLESS) |

## 设计决策

### 为什么用独立枚举而不是直接用命令类型？

SOCKS5 的 `command`、Trojan 的 `command`、VLESS 的 `command` 是各协议特有的枚举，值和语义各不相同。`form` 是协议无关的传输形态抽象，将协议特异性的命令统一为两种传输路径。

**映射关系**:
- SOCKS5: `command::connect` / `command::bind` -> `form::stream`, `command::udp_associate` -> `form::datagram`
- Trojan: `command::connect` / `command::mux` -> `form::stream`, `command::udp_associate` -> `form::datagram`
- VLESS: `command::tcp` / `command::mux` -> `form::stream`, `command::udp` -> `form::datagram`

## 使用场景

### Pipeline 路由

根据 `form` 值选择不同的转发路径：

- `stream` -> TCP tunnel 双向转发（或 mux 多路复用）
- `datagram` -> UDP relay 数据报转发

### Request 结构

每个协议的 `request` 结构都包含 `form transport` 字段，在握手阶段根据命令类型填充：

```cpp
// SOCKS5 request
struct request {
    command cmd;
    std::uint16_t destination_port;
    address destination_address;
    form transport = form::stream;  // 由握手阶段设置
};
```

> **约束**: 该枚举用于协议层和 pipeline 层，不应在传输层内部使用。

## 调用链

```
protocol/socks5/stream::handshake -> request.transport = form::stream/datagram
protocol/trojan/relay::handshake -> request.transport = form::stream/datagram
protocol/vless/format::parse_request -> request.transport = form::stream/datagram
pipeline handler -> 根据 form 选择 TCP tunnel 或 UDP relay
```
