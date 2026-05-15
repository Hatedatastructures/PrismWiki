---
title: "types — TLS 协议共享类型与常量"
source: "include/prism/protocol/tls/types.hpp"
module: "protocol"
type: api
tags: [protocol, tls, types, 常量, 记录层, ClientHello, RFC8446]
related:
  - "[[protocol/tls/signal|signal]]"
  - "[[protocol/tls/feature_bitmap|feature_bitmap]]"
  - "[[stealth/reality/scheme|reality]]"
  - "[[stealth/shadowtls/scheme|shadowtls]]"
created: 2026-05-15
updated: 2026-05-15
---

# types.hpp

> 源码: `include/prism/protocol/tls/types.hpp`
> 模块: [[protocol|protocol]] > tls

## 概述

TLS 协议共享类型与常量。定义 TLS 1.3 记录层和握手协议的通用常量，以及从 ClientHello 提取的特征结构。这些类型是中立的共享层，供 [[recognition|recognition]] 和 [[stealth|stealth]] 模块共同使用，不依赖任何具体实现。遵循 RFC 8446 (TLS 1.3) 和 RFC 5246 (TLS 1.2) 的定义。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory|memory]] | `memory::vector`、`memory::string` |
| 被依赖 | [[protocol/tls/signal|signal]] | ClientHello 解析器使用类型 |
| 被依赖 | [[protocol/tls/feature_bitmap|feature_bitmap]] | 特征位图使用 `client_hello_features` |
| 被依赖 | [[stealth/reality/handshake|reality]] | Reality 握手使用常量和类型 |
| 被依赖 | [[stealth/shadowtls/auth|shadowtls]] | ShadowTLS 认证使用特征 |

## 命名空间

`psm::protocol::tls`

---

## 记录层常量

TLS 记录层的基本参数，对应 RFC 8446 Section 5.1。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `RECORD_HEADER_LEN` | `std::size_t` | `5` | TLS 记录头长度（字节）：ContentType(1) + Version(2) + Length(2) |
| `MAX_RECORD_PAYLOAD` | `std::size_t` | `16384` | TLS 记录最大载荷长度 |

---

## Content Type 常量

TLS 记录层内容类型，对应 RFC 8446 Section 5.1。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `CONTENT_TYPE_CHANGE_CIPHER_SPEC` | `std::uint8_t` | `0x14` | ChangeCipherSpec 消息 |
| `CONTENT_TYPE_ALERT` | `std::uint8_t` | `0x15` | Alert 消息 |
| `CONTENT_TYPE_HANDSHAKE` | `std::uint8_t` | `0x16` | Handshake 消息 |
| `CONTENT_TYPE_APPLICATION_DATA` | `std::uint8_t` | `0x17` | ApplicationData 消息 |

---

## Handshake Type 常量

TLS 握手消息类型，对应 RFC 8446 Section 4。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `HANDSHAKE_TYPE_CLIENT_HELLO` | `std::uint8_t` | `0x01` | ClientHello 消息 |
| `HANDSHAKE_TYPE_SERVER_HELLO` | `std::uint8_t` | `0x02` | ServerHello 消息 |
| `HANDSHAKE_TYPE_ENCRYPTED_EXTENSIONS` | `std::uint8_t` | `0x08` | EncryptedExtensions 消息 |
| `HANDSHAKE_TYPE_CERTIFICATE` | `std::uint8_t` | `0x0B` | Certificate 消息 |
| `HANDSHAKE_TYPE_CERTIFICATE_VERIFY` | `std::uint8_t` | `0x0F` | CertificateVerify 消息 |
| `HANDSHAKE_TYPE_FINISHED` | `std::uint8_t` | `0x14` | Finished 消息 |

---

## Extension Type 常量

TLS 扩展类型，对应 RFC 8446 Section 4.2。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `EXT_SERVER_NAME` | `std::uint16_t` | `0x0000` | SNI 扩展（Server Name Indication） |
| `EXT_KEY_SHARE` | `std::uint16_t` | `0x0033` | KeyShare 扩展 |
| `EXT_SUPPORTED_VERSIONS` | `std::uint16_t` | `0x002B` | SupportedVersions 扩展 |

---

## Named Groups 常量

TLS 命名组，对应 RFC 8446 Section 4.2.7。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `NAMED_GROUP_X25519` | `std::uint16_t` | `0x001D` | X25519 曲线 |
| `NAMED_GROUP_X25519_MLKEM768` | `std::uint16_t` | `0x11EC` | X25519Kyber768 混合密钥交换 |

---

## TLS Version 常量

TLS 版本标识。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `VERSION_TLS12` | `std::uint16_t` | `0x0303` | TLS 1.2 版本号 |
| `VERSION_TLS13` | `std::uint16_t` | `0x0304` | TLS 1.3 版本号 |

---

## Cipher Suite 常量

TLS 1.3 密码套件。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `CIPHER_AES_128_GCM_SHA256` | `std::uint16_t` | `0x1301` | AES-128-GCM-SHA256 |

---

## Server Name Type 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `SERVER_NAME_TYPE_HOSTNAME` | `std::uint8_t` | `0x00` | 主机名类型（SNI） |

---

## 密钥与认证常量

Reality、AEAD 等加密相关常量。

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `REALITY_KEY_LEN` | `std::size_t` | `32` | X25519 密钥/公钥长度（字节） |
| `SESSION_ID_MAX_LEN` | `std::size_t` | `32` | TLS session_id 最大长度 |
| `AEAD_TAG_LEN` | `std::size_t` | `16` | AEAD 认证标签长度 |
| `AEAD_NONCE_LEN` | `std::size_t` | `12` | AEAD nonce 长度 |
| `AES_128_KEY_LEN` | `std::size_t` | `16` | AES-128 密钥长度 |

---

## Signature Scheme 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `SIGNATURE_SCHEME_ED25519` | `std::uint16_t` | `0x0807` | Ed25519 签名算法 |

---

## 函数: write_u8()

### 功能说明

向字节缓冲区追加 1 字节无符号整数，用于 TLS 记录层写入操作。

### 签名

```cpp
inline auto write_u8(memory::vector<std::uint8_t> &buf, std::uint8_t val) -> void;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buf` | `memory::vector<std::uint8_t> &` | 输出缓冲区 |
| `val` | `std::uint8_t` | 待写入的值 |

### 返回值

无（`void`）

### 调用（向下）

- `memory::vector::push_back()`

### 被调用（向上）

- [[stealth/reality/handshake|reality]] — Reality 握手构造 TLS 消息
- [[stealth/shadowtls/auth|shadowtls]] — ShadowTLS 认证构造 TLS 消息

### 知识域

TLS 记录层、字节序

---

## 函数: write_u16()

### 功能说明

向字节缓冲区追加 2 字节大端序无符号整数，用于 TLS 记录层写入操作。

### 签名

```cpp
inline auto write_u16(memory::vector<std::uint8_t> &buf, std::uint16_t val) -> void;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buf` | `memory::vector<std::uint8_t> &` | 输出缓冲区 |
| `val` | `std::uint16_t` | 待写入的值 |

### 返回值

无（`void`）

### 调用（向下）

- `memory::vector::push_back()`

### 被调用（向上）

- [[stealth/reality/handshake|reality]] — Reality 握手构造 TLS 消息
- [[protocol/tls/signal|signal]] — ClientHello 解析辅助

### 知识域

TLS 记录层、大端序编码

---

## 函数: write_u24()

### 功能说明

向字节缓冲区追加 3 字节大端序无符号整数，用于 TLS Handshake 消息长度字段。

### 签名

```cpp
inline auto write_u24(memory::vector<std::uint8_t> &buf, std::size_t val) -> void;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buf` | `memory::vector<std::uint8_t> &` | 输出缓冲区 |
| `val` | `std::size_t` | 待写入的值 |

### 返回值

无（`void`）

### 调用（向下）

- `memory::vector::push_back()`

### 被调用（向上）

- [[stealth/reality/handshake|reality]] — Reality 握手构造 Handshake 消息
- [[stealth/shadowtls/auth|shadowtls]] — ShadowTLS 认证构造 Handshake 消息

### 知识域

TLS Handshake 消息格式

---

## 结构体: client_hello_features

### 功能说明

从 TLS ClientHello 提取的关键特征。包含 SNI、session_id、key_share、支持的版本等特征。各方案的 `detect()` 方法基于这些特征判断置信度。该结构是中立的共享类型，供 [[recognition|recognition]] 和 [[stealth|stealth]] 共同使用。

### 签名

```cpp
struct client_hello_features {
    memory::string server_name;
    memory::vector<std::uint8_t> session_id;
    std::uint8_t session_id_len{0};
    bool has_x25519{false};
    std::array<std::uint8_t, 32> x25519_key{};
    memory::vector<std::uint16_t> versions;
    std::array<std::uint8_t, 32> random{};
    memory::vector<std::uint8_t> raw_hs_msg;
    memory::vector<std::byte> raw_record;
};
```

### 字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `server_name` | `memory::string` | 空 | SNI 服务器名称 |
| `session_id` | `memory::vector<std::uint8_t>` | 空 | session_id 原始数据 |
| `session_id_len` | `std::uint8_t` | `0` | session_id 长度（0-32） |
| `has_x25519` | `bool` | `false` | 是否存在 X25519 key_share 扩展 |
| `x25519_key` | `std::array<std::uint8_t, 32>` | 全零 | X25519 公钥（32 字节，`has_x25519` 为 `true` 时有效） |
| `versions` | `memory::vector<std::uint16_t>` | 空 | 支持的 TLS 版本列表 |
| `random` | `std::array<std::uint8_t, 32>` | 全零 | 客户端随机数（32 字节） |
| `raw_hs_msg` | `memory::vector<std::uint8_t>` | 空 | 原始握手消息（不含 TLS record header） |
| `raw_record` | `memory::vector<std::byte>` | 空 | 原始 ClientHello 记录（含 TLS record header） |

### 调用（向下）

- `memory::string`、`memory::vector` 构造（PMR 分配）

### 被调用（向上）

- [[protocol/tls/signal|signal]] — `parse_client_hello()` 填充该结构
- [[protocol/tls/feature_bitmap|feature_bitmap]] — `build_feature_bitmap()` 读取该结构
- [[stealth/reality/scheme|reality]] — Reality `detect()` 检查特征
- [[stealth/shadowtls/auth|shadowtls]] — ShadowTLS `detect()` 检查特征

### 知识域

TLS ClientHello 解析、协议指纹识别

## 相关页面

- [[protocol/tls/signal|signal]] — ClientHello 解析器
- [[protocol/tls/feature_bitmap|feature_bitmap]] — 特征位图
- [[stealth/reality/scheme|reality]] — Reality 伪装方案
- [[stealth/shadowtls/scheme|shadowtls]] — ShadowTLS 伪装方案
