---
title: "constants — Trojan 协议常量"
source: "include/prism/protocol/trojan/constants.hpp"
module: "protocol"
type: api
tags: [protocol, trojan, constants, 常量, 命令字, 地址类型]
related:
  - "[[protocol/trojan/config|config]]"
  - "[[protocol/trojan/format|format]]"
  - "[[protocol/trojan/relay|relay]]"
created: 2026-05-15
updated: 2026-05-15
---

# constants.hpp

> 源码: `include/prism/protocol/trojan/constants.hpp`
> 模块: [[protocol|protocol]] > trojan

## 概述

Trojan 协议常量定义。定义命令字和地址类型枚举。命令字用于区分 TCP 隧道连接和 UDP 关联请求，地址类型用于标识目标地址的格式。这些常量遵循 Trojan 协议规范。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/trojan/format|format]] | 格式编解码使用常量 |
| 被依赖 | [[protocol/trojan/relay|relay]] | 中继器使用常量 |
| 被依赖 | [[protocol/trojan/message|message]] | 消息结构使用常量 |

## 命名空间

`psm::protocol::trojan`

---

## 枚举: command

Trojan 协议命令字，定义客户端请求的操作类型。命令字出现在协议头部的固定位置。

| 值 | 数值 | 说明 |
|----|------|------|
| `connect` | `0x01` | 建立 TCP 隧道连接 |
| `udp_associate` | `0x03` | 建立 UDP over TLS 关联 |
| `mux` | `0x7F` | Mihomo smux 多路复用命令 |

---

## 枚举: address_type

Trojan 协议地址类型，定义目标地址的编码格式。地址类型字段用于指示后续地址数据的格式和长度。

| 值 | 数值 | 说明 |
|----|------|------|
| `ipv4` | `0x01` | IPv4 地址，占用 4 字节 |
| `domain` | `0x03` | 域名地址，格式为长度前缀加域名内容，最大 255 字节 |
| `ipv6` | `0x04` | IPv6 地址，占用 16 字节 |

## 相关页面

- [[protocol/trojan/config|config]] — 协议配置
- [[protocol/trojan/format|format]] — 格式编解码
- [[protocol/trojan/relay|relay]] — 协议中继器
