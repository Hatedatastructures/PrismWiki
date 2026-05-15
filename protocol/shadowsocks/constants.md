---
title: "constants — SS2022 协议常量"
source: "include/prism/protocol/shadowsocks/constants.hpp"
module: "protocol"
type: api
tags: [protocol, shadowsocks, constants, 常量, aead, cipher, SIP022]
related:
  - "[[protocol/shadowsocks/config|config]]"
  - "[[protocol/shadowsocks/format|format]]"
  - "[[protocol/shadowsocks/relay|relay]]"
created: 2026-05-15
updated: 2026-05-15
---

# constants.hpp

> 源码: `include/prism/protocol/shadowsocks/constants.hpp`
> 模块: [[protocol|protocol]] > shadowsocks

## 概述

SS2022 (SIP022) 协议常量定义。包括加密算法枚举、帧类型、长度常量和地址类型。包含 TCP 和 UDP 两种传输模式的常量。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/shadowsocks/format|format]] | 格式编解码使用常量 |
| 被依赖 | [[protocol/shadowsocks/relay|relay]] | 中继器使用常量 |
| 被依赖 | [[protocol/shadowsocks/message|message]] | 消息结构使用常量 |

## 命名空间

`psm::protocol::shadowsocks`

---

## 枚举: cipher_method

支持的 AEAD 加密算法。对应 SIP022 规范定义的三种加密方法。

| 值 | 说明 |
|----|------|
| `aes_128_gcm` | 2022-blake3-aes-128-gcm，16 字节密钥/salt |
| `aes_256_gcm` | 2022-blake3-aes-256-gcm，32 字节密钥/salt |
| `chacha20_poly1305` | 2022-blake3-chacha20-poly1305，32 字节密钥/salt |

---

## TCP 协议常量

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `request_type` | `std::uint8_t` | `0x00` | 请求固定头类型字节 |
| `response_type` | `std::uint8_t` | `0x01` | 响应固定头类型字节 |
| `aead_tag_len` | `std::size_t` | `16` | AEAD 认证标签长度（AES-GCM / ChaCha20） |
| `fixed_header_plain` | `std::size_t` | `11` | 固定头明文长度：type(1) + timestamp(8) + varHeaderLen(2) |
| `fixed_header_size` | `std::size_t` | `27` | 固定头密文长度：明文(11) + tag(16) |
| `kdf_context` | `std::string_view` | `"shadowsocks 2022 session subkey"` | BLAKE3 KDF 上下文（SIP022 规范统一） |
| `max_chunk_size` | `std::uint16_t` | `0x3FFF` | 数据块最大 payload 长度 |
| `length_block_size` | `std::size_t` | `18` | 加密长度块大小：2 字节长度 + 16 字节 tag |

---

## 地址类型常量

SOCKS5 兼容的地址类型标识。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `atyp_ipv4` | `std::uint8_t` | `0x01` | IPv4 地址类型 |
| `atyp_domain` | `std::uint8_t` | `0x03` | 域名地址类型 |
| `atyp_ipv6` | `std::uint8_t` | `0x04` | IPv6 地址类型 |

---

## UDP 协议常量

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `session_id_len` | `std::size_t` | `8` | UDP Session ID 长度 |
| `packet_id_len` | `std::size_t` | `8` | UDP Packet ID 长度 |
| `separate_header_len` | `std::size_t` | `16` | 分离头总长度（SessionID + PacketID） |

---

## 方法名映射

配置文件中使用的加密方法名称字符串。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `method_aes_128` | `std::string_view` | `"2022-blake3-aes-128-gcm"` | AES-128-GCM 算法标识 |
| `method_aes_256` | `std::string_view` | `"2022-blake3-aes-256-gcm"` | AES-256-GCM 算法标识 |
| `method_chacha20` | `std::string_view` | `"2022-blake3-chacha20-poly1305"` | ChaCha20-Poly1305 算法标识 |

## 相关页面

- [[protocol/shadowsocks/config|config]] — 协议配置
- [[protocol/shadowsocks/format|format]] — 格式编解码
- [[protocol/shadowsocks/relay|relay]] — 协议中继器
