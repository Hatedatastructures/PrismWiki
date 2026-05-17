---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/constants.hpp
---

# Reality Constants

TLS 1.3 记录层和握手协议的常量定义。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/constants.hpp`

## 常量定义

### TLS 记录层

| 常量 | 值 | 说明 |
|------|-----|------|
| `RECORD_HEADER_LEN` | 5 | TLS 记录头长度（ContentType + Version + Length） |
| `MAX_RECORD_PAYLOAD` | 16384 | TLS 记录最大载荷长度 |

### Content Type

| 常量 | 值 | 说明 |
|------|-----|------|
| `CONTENT_TYPE_CHANGE_CIPHER_SPEC` | 0x14 | ChangeCipherSpec |
| `CONTENT_TYPE_ALERT` | 0x15 | Alert |
| `CONTENT_TYPE_HANDSHAKE` | 0x16 | Handshake |
| `CONTENT_TYPE_APPLICATION_DATA` | 0x17 | Application Data |

### Handshake Type

| 常量 | 值 | 说明 |
|------|-----|------|
| `HANDSHAKE_TYPE_CLIENT_HELLO` | 0x01 | ClientHello |
| `HANDSHAKE_TYPE_SERVER_HELLO` | 0x02 | ServerHello |
| `HANDSHAKE_TYPE_ENCRYPTED_EXTENSIONS` | 0x08 | EncryptedExtensions |
| `HANDSHAKE_TYPE_CERTIFICATE` | 0x0B | Certificate |
| `HANDSHAKE_TYPE_CERTIFICATE_VERIFY` | 0x0F | CertificateVerify |
| `HANDSHAKE_TYPE_FINISHED` | 0x14 | Finished |

### Extension Type

| 常量 | 值 | 说明 |
|------|-----|------|
| `EXT_SERVER_NAME` | 0x0000 | server_name (SNI) |
| `EXT_KEY_SHARE` | 0x0033 | key_share |
| `EXT_SUPPORTED_VERSIONS` | 0x002B | supported_versions |

### Named Groups

| 常量 | 值 | 说明 |
|------|-----|------|
| `NAMED_GROUP_X25519` | 0x001D | X25519 ECDH |
| `NAMED_GROUP_X25519_MLKEM768` | 0x11EC | X25519 + MLKEM768 混合 |

### TLS Version

| 常量 | 值 | 说明 |
|------|-----|------|
| `VERSION_TLS12` | 0x0303 | TLS 1.2 |
| `VERSION_TLS13` | 0x0304 | TLS 1.3 |

### Cipher Suite

| 常量 | 值 | 说明 |
|------|-----|------|
| `CIPHER_AES_128_GCM_SHA256` | 0x1301 | TLS_AES_128_GCM_SHA256 |

### Signature Scheme

| 常量 | 值 | 说明 |
|------|-----|------|
| `SIGNATURE_SCHEME_ED25519` | 0x0807 | Ed25519 签名算法 |

### Reality 认证参数

| 常量 | 值 | 说明 |
|------|-----|------|
| `REALITY_KEY_LEN` | 32 | X25519 密钥/公钥长度 |
| `SESSION_ID_MAX_LEN` | 32 | session_id 最大长度 |
| `AEAD_TAG_LEN` | 16 | AEAD tag 长度 |
| `AEAD_NONCE_LEN` | 12 | AEAD nonce 长度 |
| `AES_128_KEY_LEN` | 16 | AES-128-GCM 密钥长度 |

## TLS 记录层写入工具

### write_u8

```cpp
inline auto write_u8(memory::vector<std::uint8_t> &buf, std::uint8_t val) -> void
```

写入单字节。

### write_u16

```cpp
inline auto write_u16(memory::vector<std::uint8_t> &buf, std::uint16_t val) -> void
```

写入 2 字节（大端序）。

### write_u24

```cpp
inline auto write_u24(memory::vector<std::uint8_t> &buf, std::size_t val) -> void
```

写入 3 字节（大端序）。

## 命名空间

```cpp
namespace psm::stealth::reality::tls
```

所有常量定义在 `tls` 子命名空间下。

## 调用链

- [[request]] ← 解析 ClientHello
- [[response]] ← 生成 ServerHello
- [[seal]] ← 加密传输层
- [[keygen]] ← 密钥派生