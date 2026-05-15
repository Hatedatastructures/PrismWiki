---
title: "constants — VLESS 协议常量"
source: "include/prism/protocol/vless/constants.hpp"
module: "protocol"
type: api
tags: [protocol, vless, constants, 常量, 版本号, 命令字]
related:
  - "[[protocol/vless/config|config]]"
  - "[[protocol/vless/format|format]]"
  - "[[protocol/vless/relay|relay]]"
created: 2026-05-15
updated: 2026-05-15
---

# constants.hpp

> 源码: `include/prism/protocol/vless/constants.hpp`
> 模块: [[protocol|protocol]] > vless

## 概述

VLESS 协议常量定义。定义版本号、命令字和地址类型枚举。VLESS 协议运行在 TLS 内层，本身不提供加密。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/vless/format|format]] | 格式编解码使用常量 |
| 被依赖 | [[protocol/vless/relay|relay]] | 中继器使用常量 |
| 被依赖 | [[protocol/vless/message|message]] | 消息结构使用常量 |

## 命名空间

`psm::protocol::vless`

---

## 常量: version

VLESS 协议版本号，固定为 `0x00`。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `version` | `std::uint8_t` | `0x00` | VLESS 协议版本号 |

---

## 枚举: command

VLESS 协议命令字，定义客户端请求的操作类型。

| 值 | 数值 | 说明 |
|----|------|------|
| `tcp` | `0x01` | TCP 代理 |
| `udp` | `0x02` | UDP 代理 |
| `mux` | `0x7F` | 多路复用命令（sing-box 兼容） |

---

## 枚举: address_type

VLESS 协议地址类型。注意：VLESS 的地址类型值与 Trojan/SOCKS5 不同。

| 值 | 数值 | 说明 |
|----|------|------|
| `ipv4` | `0x01` | IPv4 地址，占用 4 字节 |
| `domain` | `0x02` | 域名地址，格式为长度前缀加域名内容 |
| `ipv6` | `0x03` | IPv6 地址，占用 16 字节 |

> **注意**: VLESS 的 `domain` 地址类型为 `0x02`，而 SOCKS5/Trojan 使用 `0x03`。VLESS 的 `ipv6` 为 `0x03`，而 SOCKS5/Trojan 使用 `0x04`。

## 相关页面

- [[protocol/vless/config|config]] — 协议配置
- [[protocol/vless/format|format]] — 格式编解码
- [[protocol/vless/relay|relay]] — 协议中继器
