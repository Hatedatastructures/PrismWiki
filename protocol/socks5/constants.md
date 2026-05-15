---
title: "constants — SOCKS5 协议常量"
source: "include/prism/protocol/socks5/constants.hpp"
module: "protocol"
type: api
tags: [protocol, socks5, constants, 常量, RFC1928, RFC1929]
related:
  - "[[protocol/socks5/config|config]]"
  - "[[protocol/socks5/wire|wire]]"
  - "[[protocol/socks5/stream|stream]]"
created: 2026-05-15
updated: 2026-05-15
---

# constants.hpp

> 源码: `include/prism/protocol/socks5/constants.hpp`
> 模块: [[protocol|protocol]] > socks5

## 概述

SOCKS5 协议常量定义 (RFC 1928 / RFC 1929)。定义命令字、地址类型、认证方法和响应码。所有枚举值均为单字节无符号整数，与协议格式保持一致，可直接用于网络字节序读写。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/socks5/wire|wire]] | 线级解析使用常量 |
| 被依赖 | [[protocol/socks5/stream|stream]] | 中继器使用常量 |
| 被依赖 | [[protocol/socks5/message|message]] | 消息结构使用常量 |

## 命名空间

`psm::protocol::socks5`

---

## 枚举: command

SOCKS5 命令类型，定义客户端请求的操作。对应 RFC 1928 Section 4 的 CMD 字段。

| 值 | 数值 | 说明 |
|----|------|------|
| `connect` | `0x01` | 建立 TCP 连接（CONNECT 命令） |
| `bind` | `0x02` | 绑定端口等待反向连接（BIND 命令） |
| `udp_associate` | `0x03` | 建立 UDP 关联（UDP_ASSOCIATE 命令） |

---

## 枚举: address_type

SOCKS5 地址类型，定义目标地址的编码格式。对应 RFC 1928 Section 4 的 ATYP 字段。

| 值 | 数值 | 说明 |
|----|------|------|
| `ipv4` | `0x01` | IPv4 地址（4 字节） |
| `domain` | `0x03` | 域名地址（1 字节长度 + 域名内容） |
| `ipv6` | `0x04` | IPv6 地址（16 字节） |

---

## 枚举: auth_method

SOCKS5 认证方法，定义客户端和服务端协商的认证方式。对应 RFC 1928 Section 3 的 METHOD 字段。

| 值 | 数值 | 说明 |
|----|------|------|
| `no_auth` | `0x00` | 无需认证 |
| `gssapi` | `0x01` | GSSAPI 认证（本项目不实现） |
| `password` | `0x02` | 用户名/密码认证（RFC 1929） |
| `no_acceptable_methods` | `0xFF` | 无可接受的认证方法（协商失败） |

---

## 枚举: reply_code

SOCKS5 响应码，定义服务端返回给客户端的响应状态。对应 RFC 1928 Section 6 的 REP 字段。

| 值 | 数值 | 说明 |
|----|------|------|
| `succeeded` | `0x00` | 成功 |
| `server_failure` | `0x01` | 服务器内部错误 |
| `connection_not_allowed` | `0x02` | 连接被策略拒绝 |
| `network_unreachable` | `0x03` | 网络不可达 |
| `host_unreachable` | `0x04` | 主机不可达 |
| `connection_refused` | `0x05` | 连接被目标拒绝 |
| `ttl_expired` | `0x06` | TTL 过期 |
| `command_not_supported` | `0x07` | 不支持的命令 |
| `address_type_not_supported` | `0x08` | 不支持的地址类型 |

## 相关页面

- [[protocol/socks5/config|config]] — 协议配置
- [[protocol/socks5/wire|wire]] — 线级解析
- [[protocol/socks5/stream|stream]] — SOCKS5 中继器
