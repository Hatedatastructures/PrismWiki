---
layer: core
source: "I:/code/Prism/include/prism/protocol/vless/constants.hpp"
title: VLESS 协议常量
tags: [protocol, vless, constants, command, address-type, version]
---

# VLESS 协议常量

> 源码位置: `I:/code/Prism/include/prism/protocol/vless/constants.hpp`

## 概述

VLESS 协议常量定义，定义版本号、命令字和地址类型枚举。VLESS 协议运行在 TLS 内层，本身不提供加密。

## 命名空间

```cpp
namespace psm::protocol::vless
```

## 版本号

```cpp
constexpr std::uint8_t version = 0x00;
```

协议版本号固定为 0x00。

## 命令枚举

### command

VLESS 协议命令字枚举。

```cpp
enum class command : std::uint8_t
{
    tcp = 0x01,  // TCP 代理
    udp = 0x02,  // UDP 代理
    mux = 0x7F   // 多路复用命令（sing-box 兼容）
};
```

**命令说明**:

| 值 | 常量 | 说明 |
|---|------|------|
| 0x01 | `tcp` | TCP 代理 |
| 0x02 | `udp` | UDP 代理 |
| 0x7F | `mux` | 多路复用命令（sing-box 兼容） |

## 地址类型枚举

### address_type

VLESS 协议地址类型枚举。

```cpp
enum class address_type : std::uint8_t
{
    ipv4 = 0x01,   // IPv4 地址，占用 4 字节
    domain = 0x02, // 域名地址，长度前缀格式
    ipv6 = 0x03    // IPv6 地址，占用 16 字节
};
```

**重要**: VLESS 的地址类型值与 Trojan/SOCKS5 不同。

| 值 | 常量 | 格式 | 长度 |
|---|------|------|------|
| 0x01 | `ipv4` | 4 字节二进制 | 4 字节 |
| 0x02 | `domain` | 长度前缀 + 域名内容 | 1 + N 字节 |
| 0x03 | `ipv6` | 16 字节二进制 | 16 字节 |

## 地址类型对比

| 协议 | IPv4 | Domain | IPv6 |
|------|------|--------|------|
| VLESS | 0x01 | 0x02 | 0x03 |
| Trojan | 0x01 | 0x03 | 0x04 |
| SOCKS5 | 0x01 | 0x03 | 0x04 |

## 调用链

- [[core/protocol/vless/format|Format]] - 使用这些常量解析协议头部
- [[core/protocol/vless/relay|Relay]] - 中继器检查命令类型
- [[core/protocol/vless/config|Config]] - 配置控制允许的命令