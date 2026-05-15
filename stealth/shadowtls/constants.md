---
title: "constants — ShadowTLS v3 协议常量"
source: "include/prism/stealth/shadowtls/constants.hpp"
module: "stealth"
submodule: "shadowtls"
type: api
tags: [stealth, shadowtls, constants, 常量]
created: 2026-05-15
updated: 2026-05-15
---

# constants.hpp

> 源码: `include/prism/stealth/shadowtls/constants.hpp`
> 模块: [[stealth|stealth]] > [[stealth/shadowtls|shadowtls]]

## 概述

ShadowTLS v3 协议常量。定义 ShadowTLS v3 协议中使用的固定常数值，这些常量与 TLS 1.3 记录层格式和 ShadowTLS 认证机制相关。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[stealth/shadowtls/auth|auth]] | 认证函数使用常量 |
| 被依赖 | [[stealth/shadowtls/handshake|handshake]] | 握手函数使用常量 |

## 命名空间

`psm::stealth::shadowtls`

---

## TLS 记录层常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| tls_header_size | std::size_t | 5 | TLS 记录头长度 |
| tls_random_size | std::size_t | 32 | TLS Random 长度 |
| tls_session_id_size | std::size_t | 32 | ShadowTLS 要求的 SessionID 长度 |
| hmac_size | std::size_t | 4 | HMAC 标签长度（4 字节） |

### 说明

- **tls_header_size**: TLS 记录层头固定 5 字节（1 字节类型 + 2 字节版本 + 2 字节长度）
- **tls_random_size**: TLS ClientHello/ServerHello 中的 Random 字段固定 32 字节
- **tls_session_id_size**: ShadowTLS 要求的 SessionID 长度固定 32 字节
- **hmac_size**: ShadowTLS 使用的 HMAC 标签长度固定 4 字节

---

## TLS 内容类型

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| content_type_handshake | std::uint8_t | 0x16 | TLS 握手消息 |
| content_type_application_data | std::uint8_t | 0x17 | TLS 应用数据 |

### 说明

- **content_type_handshake**: TLS 记录层内容类型，表示握手消息
- **content_type_application_data**: TLS 记录层内容类型，表示应用数据

---

## TLS 握手类型

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| handshake_type_client_hello | std::uint8_t | 0x01 | ClientHello 消息 |
| handshake_type_server_hello | std::uint8_t | 0x02 | ServerHello 消息 |

### 说明

- **handshake_type_client_hello**: TLS 握手消息类型，表示 ClientHello
- **handshake_type_server_hello**: TLS 握手消息类型，表示 ServerHello

---

## TLS 1.3 版本号

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| tls_version_1_3 | std::uint16_t | 0x0304 | TLS 1.3 版本号 |

### 说明

- **tls_version_1_3**: TLS 1.3 协议版本号，用于 ClientHello 和 ServerHello

---

## TLS 扩展类型

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| extension_supported_versions | std::uint16_t | 43 | supported_versions 扩展 |

### 说明

- **extension_supported_versions**: TLS 扩展类型，用于声明支持的 TLS 版本

---

## SessionID 中 HMAC 的位置

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| session_id_length_index | std::size_t | 43 | ClientHello 中 SessionID 长度字节的偏移 |

### 说明

- **session_id_length_index**: ClientHello 中 SessionID 长度字节的偏移位置
- 计算方式: TLS Header(5) + Handshake Header(4) + SessionID Length(1) = 10
- SessionID 32 字节，HMAC 在最后 4 字节

---

## 数据帧头大小

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| tls_hmac_header_size | std::size_t | 9 | 数据帧头大小（TLS Header 5 + HMAC 4） |

### 说明

- **tls_hmac_header_size**: 数据帧头大小，包含 TLS 记录头和 HMAC 标签
- 计算方式: tls_header_size(5) + hmac_size(4) = 9

---

## 使用场景

### 认证阶段

- `tls_session_id_size`: 验证 SessionID 长度
- `hmac_size`: 提取 SessionID 中的 HMAC 标签
- `session_id_length_index`: 定位 SessionID 长度字节

### 握手阶段

- `tls_header_size`: 解析 TLS 记录头
- `tls_random_size`: 提取 ServerRandom
- `content_type_handshake`: 识别握手消息
- `handshake_type_client_hello`: 识别 ClientHello
- `handshake_type_server_hello`: 识别 ServerHello
- `tls_version_1_3`: 验证 TLS 版本

### 数据帧阶段

- `tls_hmac_header_size`: 解析数据帧头
- `hmac_size`: 提取数据帧中的 HMAC 标签
- `content_type_application_data`: 识别应用数据

---

## 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — 常量值遵循 RFC 8446 定义
- [[ref/protocol/tls-sessionid|TLS SessionID]] — SessionID 相关常量用于认证

## 相关链接

- [[stealth/shadowtls/auth|auth]] — 认证函数，使用这些常量
- [[stealth/shadowtls/handshake|handshake]] — 握手函数，使用这些常量
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 协议
- [[ref/protocol/tls-sessionid|TLS SessionID]] — TLS SessionID 字段
