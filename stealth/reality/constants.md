---
title: "constants — Reality TLS 常量定义"
source: "include/prism/stealth/reality/constants.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, constants, 常量, tls]
created: 2026-05-15
updated: 2026-05-15
---

# constants.hpp

> 源码: `include/prism/stealth/reality/constants.hpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

Reality TLS 常量定义。定义 TLS 1.3 记录层和握手协议的常量，用于 ClientHello 解析、ServerHello 生成和加密记录处理。这些常量遵循 RFC 8446 (TLS 1.3) 和 RFC 5246 (TLS 1.2) 的定义。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | 使用 memory::vector PMR 容器 |
| 被依赖 | [[stealth/reality/auth|auth]] | 认证函数使用常量 |
| 被依赖 | [[stealth/reality/handshake|handshake]] | 握手函数使用常量 |
| 被依赖 | [[stealth/reality/keygen|keygen]] | 密钥生成使用常量 |
| 被依赖 | [[stealth/reality/request|request]] | 请求解析使用常量 |
| 被依赖 | [[stealth/reality/response|response]] | 响应生成使用常量 |
| 被依赖 | [[stealth/reality/seal|seal]] | 加密传输层使用常量 |

## 命名空间

`psm::stealth::reality::tls`

---

## TLS 记录层常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| RECORD_HEADER_LEN | std::size_t | 5 | TLS 记录头长度（字节）：ContentType(1) + Version(2) + Length(2) |
| MAX_RECORD_PAYLOAD | std::size_t | 16384 | TLS 记录最大载荷长度 |

---

## Content Type 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| CONTENT_TYPE_CHANGE_CIPHER_SPEC | std::uint8_t | 0x14 | ChangeCipherSpec 消息 |
| CONTENT_TYPE_ALERT | std::uint8_t | 0x15 | Alert 消息 |
| CONTENT_TYPE_HANDSHAKE | std::uint8_t | 0x16 | Handshake 消息 |
| CONTENT_TYPE_APPLICATION_DATA | std::uint8_t | 0x17 | Application Data 消息 |

---

## Handshake Type 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| HANDSHAKE_TYPE_CLIENT_HELLO | std::uint8_t | 0x01 | ClientHello 消息 |
| HANDSHAKE_TYPE_SERVER_HELLO | std::uint8_t | 0x02 | ServerHello 消息 |
| HANDSHAKE_TYPE_ENCRYPTED_EXTENSIONS | std::uint8_t | 0x08 | EncryptedExtensions 消息 |
| HANDSHAKE_TYPE_CERTIFICATE | std::uint8_t | 0x0B | Certificate 消息 |
| HANDSHAKE_TYPE_CERTIFICATE_VERIFY | std::uint8_t | 0x0F | CertificateVerify 消息 |
| HANDSHAKE_TYPE_FINISHED | std::uint8_t | 0x14 | Finished 消息 |

---

## Extension Type 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| EXT_SERVER_NAME | std::uint16_t | 0x0000 | SNI 扩展 |
| EXT_KEY_SHARE | std::uint16_t | 0x0033 | Key Share 扩展 |
| EXT_SUPPORTED_VERSIONS | std::uint16_t | 0x002B | Supported Versions 扩展 |

---

## Named Groups 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| NAMED_GROUP_X25519 | std::uint16_t | 0x001D | X25519 密钥交换 |
| NAMED_GROUP_X25519_MLKEM768 | std::uint16_t | 0x11EC | X25519+ML-KEM-768 混合密钥交换 |

---

## TLS Version 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| VERSION_TLS12 | std::uint16_t | 0x0303 | TLS 1.2 版本号 |
| VERSION_TLS13 | std::uint16_t | 0x0304 | TLS 1.3 版本号 |

---

## Cipher Suite 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| CIPHER_AES_128_GCM_SHA256 | std::uint16_t | 0x1301 | AES-128-GCM-SHA256 密码套件 |

---

## Server Name Type 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| SERVER_NAME_TYPE_HOSTNAME | std::uint8_t | 0x00 | 主机名类型 |

---

## Reality 认证常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| REALITY_KEY_LEN | std::size_t | 32 | X25519 密钥/公钥长度 |
| SESSION_ID_MAX_LEN | std::size_t | 32 | TLS session_id 最大长度 |
| AEAD_TAG_LEN | std::size_t | 16 | AEAD tag 长度 |
| AEAD_NONCE_LEN | std::size_t | 12 | AEAD nonce 长度 |
| AES_128_KEY_LEN | std::size_t | 16 | AES-128-GCM 密钥长度 |

---

## Signature Scheme 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|-----|------|
| SIGNATURE_SCHEME_ED25519 | std::uint16_t | 0x0807 | Ed25519 签名算法 |

---

## 工具函数

### write_u8()

> 源码: `include/prism/stealth/reality/constants.hpp:67`

写入 8 位无符号整数到缓冲区。

```cpp
inline auto write_u8(memory::vector<std::uint8_t> &buf, std::uint8_t val) -> void;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buf | memory::vector\<std::uint8_t\>& | 目标缓冲区（追加写入） |
| val | std::uint8_t | 要写入的 8 位值 |

#### 返回值

`void` — 无返回值

#### 调用链

**被调用**（向上）:
- [[stealth/reality/request|request]] — 构造 ClientHello 时写入单字节字段（如 ContentType、HandshakeType、ServerNameType）
- [[stealth/reality/response|response]] — 构造 ServerHello 时写入单字节字段

---

### write_u16()

> 源码: `include/prism/stealth/reality/constants.hpp:72`

写入 16 位无符号整数到缓冲区（大端序）。

```cpp
inline auto write_u16(memory::vector<std::uint8_t> &buf, std::uint16_t val) -> void;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buf | memory::vector\<std::uint8_t\>& | 目标缓冲区（追加写入） |
| val | std::uint16_t | 要写入的 16 位值（主机序，函数内部转大端序） |

#### 返回值

`void` — 无返回值

#### 调用链

**被调用**（向上）:
- [[stealth/reality/request|request]] — 构造 ClientHello 时写入 ExtensionType、NamedGroup、CipherSuite 等 16 位字段
- [[stealth/reality/response|response]] — 构造 ServerHello 时写入 Version、CipherSuite 等 16 位字段

---

### write_u24()

> 源码: `include/prism/stealth/reality/constants.hpp:78`

写入 24 位无符号整数到缓冲区（大端序）。用于 TLS Handshake 层的 Length 字段。

```cpp
inline auto write_u24(memory::vector<std::uint8_t> &buf, std::size_t val) -> void;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buf | memory::vector\<std::uint8_t\>& | 目标缓冲区（追加写入） |
| val | std::size_t | 要写入的 24 位值（仅低 24 位有效） |

#### 返回值

`void` — 无返回值

#### 调用链

**被调用**（向上）:
- [[stealth/reality/request|request]] — 构造 Handshake 头的 Length 字段
- [[stealth/reality/response|response]] — 构造 Handshake 头的 Length 字段

---

## 使用场景

### ClientHello 解析

- `RECORD_HEADER_LEN`: 解析 TLS 记录头
- `CONTENT_TYPE_HANDSHAKE`: 识别握手消息
- `HANDSHAKE_TYPE_CLIENT_HELLO`: 识别 ClientHello
- `EXT_SERVER_NAME`: 提取 SNI 扩展
- `EXT_KEY_SHARE`: 提取 key_share 扩展
- `EXT_SUPPORTED_VERSIONS`: 提取 supported_versions 扩展
- `NAMED_GROUP_X25519`: 识别 X25519 公钥

### ServerHello 生成

- `CONTENT_TYPE_HANDSHAKE`: 构造握手消息
- `HANDSHAKE_TYPE_SERVER_HELLO`: 构造 ServerHello
- `VERSION_TLS13`: 设置 TLS 版本
- `CIPHER_AES_128_GCM_SHA256`: 设置密码套件

### 加密记录处理

- `AEAD_TAG_LEN`: 计算加密记录长度
- `AEAD_NONCE_LEN`: 生成 nonce
- `AES_128_KEY_LEN`: 使用密钥

### Reality 认证

- `REALITY_KEY_LEN`: X25519 密钥长度
- `SESSION_ID_MAX_LEN`: SessionID 长度
- `SIGNATURE_SCHEME_ED25519`: Ed25519 签名

---

## 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — 常量值遵循 RFC 8446 定义
- [[ref/protocol/tls-extensions|TLS 扩展]] — Extension Type 常量对应 TLS 扩展

## 相关链接

- [[stealth/reality/auth|auth]] — Reality 认证，使用这些常量
- [[stealth/reality/handshake|handshake]] — Reality 握手，使用这些常量
- [[stealth/reality/keygen|keygen]] — Reality 密钥生成，使用这些常量
- [[stealth/reality/request|request]] — Reality 请求解析，使用这些常量
- [[stealth/reality/response|response]] — Reality 响应生成，使用这些常量
- [[stealth/reality/seal|seal]] — Reality 加密传输层，使用这些常量
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 协议
