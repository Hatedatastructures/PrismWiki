---
title: "response — TLS 1.3 ServerHello 生成器"
source: "include/prism/stealth/reality/response.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, response, serverhello, 生成]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/reality/handshake|handshake]]"
  - "[[stealth/reality/keygen|keygen]]"
  - "[[stealth/reality/request|request]]"
  - "[[stealth/reality/auth|auth]]"
  - "[[crypto/aead|aead]]"
  - "[[stealth/reality/constants|constants]]"
---

# response.hpp

> 源码: `include/prism/stealth/reality/response.hpp`
> 实现: `src/prism/stealth/reality/response.cpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

TLS 1.3 ServerHello 生成器。生成 Reality 协议所需的 TLS 1.3 服务端握手消息，包括 ServerHello、ChangeCipherSpec（兼容性）和加密的握手记录，用于构造完整的 TLS 握手响应发送给客户端。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[fault/code|code]] | 使用 fault::code 错误码 |
| 依赖 | [[memory/container|container]] | 使用 memory::vector PMR 容器 |
| 依赖 | [[stealth/reality/request|request]] | 使用 client_hello_info 结构 |
| 依赖 | [[stealth/reality/keygen|keygen]] | 使用 key_material 和 compute_finished_verify_data |
| 依赖 | [[crypto/aead|aead]] | 使用 AEAD 加密 |
| 依赖 | [[crypto/hkdf|hkdf]] | 使用 HMAC-SHA512 生成证书签名 |
| 依赖 | [[crypto/x25519|ed25519]] | 使用 Ed25519 密钥对生成和签名 |
| 依赖 | [[stealth/reality/constants|constants]] | 使用协议常量 |
| 被依赖 | [[stealth/reality/handshake|handshake]] | 握手流程中生成响应 |

## 命名空间

`psm::stealth::reality`

---

## 结构体: server_hello_result

> 源码: `include/prism/stealth/reality/response.hpp:30`

### 概述

ServerHello 生成结果。包含 TLS 握手响应的各阶段消息和记录。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| server_hello_msg | memory::vector\<uint8_t\> | ServerHello 握手消息（含 handshake header） |
| server_hello_record | memory::vector\<uint8_t\> | ServerHello TLS 记录（含 record header） |
| change_cipher_spec_record | memory::vector\<uint8_t\> | ChangeCipherSpec 兼容性记录（6 字节） |
| encrypted_handshake_record | memory::vector\<uint8_t\> | 加密后的握手记录（EncryptedExtensions + Certificate + CertificateVerify + Finished） |
| encrypted_handshake_plaintext | memory::vector\<uint8_t\> | 加密前握手记录明文 |

---

## 函数: generate_server_hello()

> 源码: `include/prism/stealth/reality/response.hpp:52`
> 实现: `src/prism/stealth/reality/response.cpp:297`

### 功能

生成 ServerHello 及完整握手响应。构造 ServerHello 消息、ChangeCipherSpec 兼容性记录和加密的握手记录（含 EncryptedExtensions、Certificate、CertificateVerify 和 Finished）。当 `auth_key` 非空时，使用 Reality 合成的 Ed25519 证书（HMAC-SHA512 签名）；否则使用 dest_certificate。

### 签名

```cpp
[[nodiscard]] auto generate_server_hello(
    const client_hello_info &client_hello,
    std::span<const std::uint8_t> server_ephemeral_public,
    const key_material &handshake_keys,
    std::span<const std::uint8_t> dest_certificate,
    std::span<const std::uint8_t> client_hello_msg,
    std::span<const std::uint8_t> auth_key = {})
    -> std::pair<fault::code, server_hello_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| client_hello | const client_hello_info& | 解析后的 ClientHello 信息 |
| server_ephemeral_public | std::span<const std::uint8_t> | 服务端临时 X25519 公钥（32 字节） |
| handshake_keys | const key_material& | 握手阶段密钥材料（用于 Finished 和加密） |
| dest_certificate | std::span<const std::uint8_t> | 目标网站的 DER 格式证书（auth_key 为空时使用） |
| client_hello_msg | std::span<const std::uint8_t> | 完整的 ClientHello 消息字节（用于 transcript hash） |
| auth_key | std::span<const std::uint8_t> | HKDF 派生的认证密钥，用于签名（默认空） |

### 返回值

`std::pair<fault::code, server_hello_result>` — 错误码和生成结果

### 调用（向下）

- `build_server_hello_body` — 构造 ServerHello 消息体
- `make_handshake_message` — 包装 handshake 消息头
- make_tls_record() — 构造 TLS 记录
- `build_encrypted_extensions` — 构造 EncryptedExtensions
- `build_certificate` — 构造 Certificate 消息（调用 `generate_reality_certificate`）
- `build_certificate_verify` — 构造 CertificateVerify（Ed25519 签名）
- [[stealth/reality/keygen|compute_finished_verify_data]] — 计算 Finished verify_data
- encrypt_tls_record() — 加密握手记录
- [[crypto/aead|aead_context::seal]] — AEAD 加密

### 被调用（向上）

- [[stealth/reality/handshake|handshake]] — 握手流程中生成响应

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 ServerHello 结构
- [[ref/protocol/tls-serverhello|TLS ServerHello]] — TLS ServerHello 消息
- [[ref/crypto/ed25519|Ed25519]] — Ed25519 证书签名

### 流程

1. 构造 ServerHello 消息体（版本、随机数、session_id、cipher_suite、extensions）
2. 包装为 handshake 消息和 TLS 记录
3. 构造 ChangeCipherSpec 兼容性记录
4. 构造 EncryptedExtensions（空扩展）
5. 构造 Certificate 消息（Reality 合成证书或 dest 证书）
6. 计算 CertificateVerify 的 transcript hash
7. 构造 CertificateVerify（Ed25519 签名）
8. 计算 Finished 的 transcript hash
9. 计算 Finished verify_data
10. 拼接明文握手记录
11. 使用 handshake 密钥加密握手记录
12. 返回所有消息和记录

---

## 函数: make_tls_record()

> 源码: `include/prism/stealth/reality/response.hpp:69`
> 实现: `src/prism/stealth/reality/response.cpp:32`

### 功能

构造 TLS 记录。将载荷数据包装为标准 TLS 记录格式：`[ContentType 1B][Version 2B][Length 2B][Payload]`。版本固定为 TLS 1.2（0x0303）。

### 签名

```cpp
[[nodiscard]] auto make_tls_record(std::uint8_t content_type,
                                   std::span<const std::uint8_t> payload)
    -> memory::vector<std::uint8_t>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| content_type | std::uint8_t | 内容类型（如 0x16=Handshake, 0x17=ApplicationData） |
| payload | std::span<const std::uint8_t> | 记录载荷 |

### 返回值

`memory::vector<std::uint8_t>` — 编码后的 TLS 记录字节（含 5 字节 record header）

### 调用（向下）

- 无（纯函数）

### 被调用（向上）

- generate_server_hello() — 生成 ServerHello 记录
- encrypt_tls_record() — 加密后包装为 TLS 记录

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 记录层格式

---

## 函数: encrypt_tls_record()

> 源码: `include/prism/stealth/reality/response.hpp:84`
> 实现: `src/prism/stealth/reality/response.cpp:45`

### 功能

加密 TLS 1.3 记录。使用 AEAD 加密明文数据，构造加密后的 TLS 记录。nonce 由 IV 和序列号按字节异或生成，明文后追加 content type 作为 inner_content_type，加密后追加 AEAD tag。AAD 为 ApplicationData 类型的 TLS 记录头。

### 签名

```cpp
[[nodiscard]] auto encrypt_tls_record(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> iv,
    std::uint64_t sequence,
    std::uint8_t content_type,
    std::span<const std::uint8_t> plaintext)
    -> std::pair<fault::code, memory::vector<std::uint8_t>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| key | std::span<const std::uint8_t> | 加密密钥（AES-128，16 字节） |
| iv | std::span<const std::uint8_t> | 初始化向量（12 字节） |
| sequence | std::uint64_t | 序列号，用于生成 nonce（与 IV 异或） |
| content_type | std::uint8_t | 内容类型（追加到明文后，如 0x16=Handshake） |
| plaintext | std::span<const std::uint8_t> | 明文数据 |

### 返回值

`std::pair<fault::code, memory::vector<std::uint8_t>>` — 错误码和加密后的密文（含 record header，ContentType 为 ApplicationData）

### 调用（向下）

- [[crypto/aead|aead_context]] — AEAD 加密（AES-128-GCM）
- make_tls_record() — 包装为 TLS 记录

### 被调用（向上）

- generate_server_hello() — 加密握手记录
- [[stealth/reality/handshake|handshake]] — 握手中重算 Finished 后加密

### 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-128-GCM 认证加密
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 加密记录格式

### 流程

1. 拼接明文 + content_type 作为 inner
2. 计算 nonce = IV XOR sequence（大端序，后 8 字节）
3. 构造 AAD = ApplicationData TLS 记录头
4. AEAD seal 加密 inner
5. 包装为 ApplicationData TLS 记录
