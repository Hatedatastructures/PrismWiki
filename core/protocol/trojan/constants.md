---
layer: core
source: "I:/code/Prism/include/prism/protocol/trojan/constants.hpp"
---

# Trojan 协议常量

> 源码位置: `I:/code/Prism/include/prism/protocol/trojan/constants.hpp`

## 概述

Trojan 协议常量定义，定义命令字和地址类型枚举。命令字用于区分 TCP 隧道连接和 UDP 关联请求，地址类型用于标识目标地址的格式。

## 命名空间

```cpp
namespace psm::protocol::trojan
```

## 命令枚举

### command

Trojan 协议命令字枚举，用于指示客户端请求的操作类型。

```cpp
enum class command : std::uint8_t
{
    connect = 0x01,       // 建立 TCP 隧道连接
    udp_associate = 0x03, // 建立 UDP over TLS 关联
    mux = 0x7f            // Mihomo smux 多路复用命令
};
```

**命令说明**:

| 值 | 常量 | 说明 |
|---|------|------|
| 0x01 | `connect` | 建立 TCP 隧道连接 |
| 0x03 | `udp_associate` | 建立 UDP over TLS 关联 |
| 0x7f | `mux` | Mihomo smux 多路复用命令 |

## 地址类型枚举

### address_type

Trojan 协议地址类型枚举，指示后续地址数据的格式和长度。

```cpp
enum class address_type : std::uint8_t
{
    ipv4 = 0x01,   // IPv4 地址，占用 4 字节
    domain = 0x03, // 域名地址，长度前缀格式，最大 255 字节
    ipv6 = 0x04    // IPv6 地址，占用 16 字节
};
```

**地址类型说明**:

| 值 | 常量 | 格式 | 长度 |
|---|------|------|------|
| 0x01 | `ipv4` | 4 字节二进制 | 4 字节 |
| 0x03 | `domain` | 长度前缀 + 域名内容 | 1 + N 字节 |
| 0x04 | `ipv6` | 16 字节二进制 | 16 字节 |

## 调用链

- [[core/protocol/trojan/format|Format]] - 使用这些常量解析协议头部
- [[core/protocol/trojan/relay|Relay]] - 中继器检查命令类型
- [[core/protocol/trojan/config|Config]] - 配置控制允许的命令