---
layer: core
source: I:/code/Prism/include/prism/protocol/common/form.hpp
title: 传输形态枚举
---

# 传输形态枚举

定义传输层的数据传输形态，用于区分 TCP 流式传输和 UDP 数据报传输。

## 源码位置

`I:/code/Prism/include/prism/protocol/common/form.hpp`

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

| 值 | 说明 | 对应协议 |
|---|---|---|
| `stream` | 可靠有序字节流 | TCP，SOCKS5 CONNECT，HTTP CONNECT |
| `datagram` | 不可靠数据报 | UDP，SOCKS5 UDP_ASSOCIATE |

## 使用场景

### SOCKS5 命令映射

- [[core/protocol/socks5/constants]] `command::connect` -> `form::stream`
- [[core/protocol/socks5/constants]] `command::udp_associate` -> `form::datagram`
- [[core/protocol/socks5/constants]] `command::bind` -> `form::stream`（控制面）

### Pipeline 路由

根据 `form` 值选择不同的转发路径：

- `stream` -> TCP tunnel 双向转发
- `datagram` -> UDP relay 数据报转发

## 调用链

```
protocol/socks5/stream::handshake -> request.transport = form::stream/datagram
agent/worker/worker -> 根据 form 选择 TCP tunnel 或 UDP relay
```

## 设计原则

- 协议层专用，不应在传输层内部使用
- 单字节枚举值，便于网络传输和序列化