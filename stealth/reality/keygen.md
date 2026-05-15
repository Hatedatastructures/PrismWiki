---
title: "keygen — TLS 1.3 密钥调度"
source: "include/prism/stealth/reality/keygen.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, keygen, 密钥派生, hkdf, tls13]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/reality/handshake|handshake]]"
  - "[[stealth/reality/seal|seal]]"
  - "[[stealth/reality/response|response]]"
  - "[[crypto/hkdf|hkdf]]"
  - "[[stealth/reality/constants|constants]]"
---

# keygen.hpp

> 源码: `include/prism/stealth/reality/keygen.hpp`
> 实现: `src/prism/stealth/reality/keygen.cpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

TLS 1.3 密钥调度。实现 RFC 8446 Section 7 的 TLS 1.3 密钥派生流程，从 ECDHE 共享密钥派生握手流量密钥和应用流量密钥。Reality 协议使用自定义的 X25519 共享密钥替代标准 TLS ECDHE 结果，密钥调度算法与标准 TLS 1.3 完全一致。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[fault/code|code]] | 使用 fault::code 错误码 |
| 依赖 | [[crypto/hkdf|hkdf]] | 使用 HKDF-Extract、HKDF-Expand-Label |
| 依赖 | [[crypto/hkdf|crypto::sha256]] | 使用 SHA-256 哈希 |
| 依赖 | [[crypto/hkdf|crypto::hmac_sha256]] | 使用 HMAC-SHA256 |
| 依赖 | [[stealth/reality/constants|constants]] | 使用协议常量（AES_128_KEY_LEN、AEAD_NONCE_LEN 等） |
| 被依赖 | [[stealth/reality/handshake|handshake]] | 握手流程中派生密钥 |
| 被依赖 | [[stealth/reality/response|response]] | 使用 key_material 和 compute_finished_verify_data |
| 被依赖 | [[stealth/reality/seal|seal]] | 使用 key_material 初始化加密上下文 |

## 命名空间

`psm::stealth::reality`

---

## 结构体: key_material

> 源码: `include/prism/stealth/reality/keygen.hpp:31`

### 概述

TLS 1.3 密钥材料。包含握手阶段和应用阶段的全部加密密钥和 IV，以及 Finished 验证密钥和主密钥。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| server_handshake_key | std::array\<uint8_t, 16\> | 握手阶段服务端加密密钥（AES-128） |
| server_handshake_iv | std::array\<uint8_t, 12\> | 握手阶段服务端 IV（AEAD nonce） |
| client_handshake_key | std::array\<uint8_t, 16\> | 握手阶段客户端加密密钥（AES-128） |
| client_handshake_iv | std::array\<uint8_t, 12\> | 握手阶段客户端 IV（AEAD nonce） |
| server_app_key | std::array\<uint8_t, 16\> | 应用阶段服务端加密密钥（AES-128） |
| server_app_iv | std::array\<uint8_t, 12\> | 应用阶段服务端 IV（AEAD nonce） |
| client_app_key | std::array\<uint8_t, 16\> | 应用阶段客户端加密密钥（AES-128） |
| client_app_iv | std::array\<uint8_t, 12\> | 应用阶段客户端 IV（AEAD nonce） |
| server_finished_key | std::array\<uint8_t, 32\> | 服务端 Finished 验证密钥（HMAC-SHA256） |
| master_secret | std::array\<uint8_t, 32\> | 主密钥 |

---

## 函数: derive_handshake_keys()

> 源码: `include/prism/stealth/reality/keygen.hpp:53`
> 实现: `src/prism/stealth/reality/keygen.cpp:14`

### 功能

派生握手阶段密钥。按照 RFC 8446 Section 7 的密钥调度流程，从共享密钥和 ClientHello/ServerHello 消息派生握手流量密钥、IV 和 Finished 验证密钥。这是 TLS 1.3 密钥调度的核心函数。

### 签名

```cpp
[[nodiscard]] auto derive_handshake_keys(constspan shared_secret, constspan client_hello_msg, constspan server_hello_msg)
    -> std::pair<fault::code, key_material>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| shared_secret | constspan (std::span<const uint8_t>) | ECDHE 共享密钥（X25519 输出，32 字节） |
| client_hello_msg | constspan | 完整的 ClientHello handshake 消息字节 |
| server_hello_msg | constspan | 完整的 ServerHello handshake 消息字节 |

### 返回值

`std::pair<fault::code, key_material>` — 错误码和派生出的密钥材料

### 调用（向下）

- [[crypto/hkdf|hkdf_extract]] — HKDF-Extract（early_secret、handshake_secret、master_secret）
- [[crypto/hkdf|hkdf_expand_label]] — HKDF-Expand-Label（derived、c hs traffic、s hs traffic、key、iv、finished）
- [[crypto/hkdf|crypto::sha256]] — 计算 hello_hash 和 empty_hash

### 被调用（向上）

- [[stealth/reality/handshake|handshake]] — 握手流程中派生握手密钥

### 知识域

- [[ref/crypto/hkdf|HKDF]] — HKDF 密钥派生（RFC 5869）
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 密钥调度（RFC 8446 Section 7）

### 流程

1. `early_secret = HKDF-Extract(salt=0^32, IKM=0^32)`
2. `empty_hash = SHA-256("")`
3. `derived_secret = HKDF-Expand-Label(early_secret, "derived", empty_hash, 32)`
4. `handshake_secret = HKDF-Extract(derived_secret, shared_secret)`
5. `hello_hash = SHA-256(client_hello_msg || server_hello_msg)`
6. `c_hs_traffic = HKDF-Expand-Label(handshake_secret, "c hs traffic", hello_hash, 32)`
7. `s_hs_traffic = HKDF-Expand-Label(handshake_secret, "s hs traffic", hello_hash, 32)`
8. `server_handshake_key = HKDF-Expand-Label(s_hs_traffic, "key", {}, 16)`
9. `server_handshake_iv = HKDF-Expand-Label(s_hs_traffic, "iv", {}, 12)`
10. `client_handshake_key = HKDF-Expand-Label(c_hs_traffic, "key", {}, 16)`
11. `client_handshake_iv = HKDF-Expand-Label(c_hs_traffic, "iv", {}, 12)`
12. `derived_master = HKDF-Expand-Label(handshake_secret, "derived", empty_hash, 32)`
13. `master_secret = HKDF-Extract(derived_master, 0^32)`
14. `server_finished_key = HKDF-Expand-Label(s_hs_traffic, "finished", {}, 32)`

---

## 函数: derive_application_keys()

> 源码: `include/prism/stealth/reality/keygen.hpp:65`
> 实现: `src/prism/stealth/reality/keygen.cpp:111`

### 功能

派生应用数据密钥。从主密钥和服务端 Finished 哈希派生应用流量密钥和 IV，结果写入 keys 参数的应用阶段字段。需要在 Finished 交换完成后调用。

### 签名

```cpp
[[nodiscard]] auto derive_application_keys(constspan master_secret, constspan server_finished_hash, key_material &keys) -> fault::code;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| master_secret | constspan | 主密钥（32 字节） |
| server_finished_hash | constspan | 服务端 Finished 消息的 transcript 哈希（32 字节） |
| keys | key_material& | 密钥材料，应用阶段字段将被填充 |

### 返回值

`fault::code` — 错误码，成功时为 success

### 调用（向下）

- [[crypto/hkdf|hkdf_expand_label]] — HKDF-Expand-Label（s ap traffic、c ap traffic、key、iv）

### 被调用（向上）

- [[stealth/reality/handshake|handshake]] — 握手流程中派生应用密钥（Finished 交换后）

### 知识域

- [[ref/crypto/hkdf|HKDF]] — HKDF 密钥派生
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 应用密钥派生

### 流程

1. `s_ap_traffic = HKDF-Expand-Label(master_secret, "s ap traffic", server_finished_hash, 32)`
2. `c_ap_traffic = HKDF-Expand-Label(master_secret, "c ap traffic", server_finished_hash, 32)`
3. `server_app_key = HKDF-Expand-Label(s_ap_traffic, "key", {}, 16)`
4. `server_app_iv = HKDF-Expand-Label(s_ap_traffic, "iv", {}, 12)`
5. `client_app_key = HKDF-Expand-Label(c_ap_traffic, "key", {}, 16)`
6. `client_app_iv = HKDF-Expand-Label(c_ap_traffic, "iv", {}, 12)`

---

## 函数: compute_finished_verify_data()

> 源码: `include/prism/stealth/reality/keygen.hpp:75`
> 实现: `src/prism/stealth/reality/keygen.cpp:165`

### 功能

计算 Finished verify_data。使用 finished_key 和 transcript_hash 计算 TLS 1.3 Finished 消息的验证数据。实现为 `HMAC-SHA256(finished_key, transcript_hash)`。

### 签名

```cpp
[[nodiscard]] auto compute_finished_verify_data(constspan finished_key, constspan transcript_hash)
    -> std::array<std::uint8_t, crypto::SHA256_LEN>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| finished_key | constspan | Finished 验证密钥（32 字节） |
| transcript_hash | constspan | 握手消息的 transcript 哈希（32 字节） |

### 返回值

`std::array<std::uint8_t, crypto::SHA256_LEN>` — 计算出的 verify_data（32 字节）

### 调用（向下）

- [[crypto/hkdf|crypto::hmac_sha256]] — HMAC-SHA256 计算

### 被调用（向上）

- [[stealth/reality/handshake|handshake]] — 握手流程中计算 Finished（derive_and_encrypt_finished）
- [[stealth/reality/response|generate_server_hello]] — 生成 ServerHello 时计算 Finished

### 知识域

- [[ref/crypto/hmac-sha256|HMAC-SHA256]] — HMAC-SHA256 消息认证码
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 Finished 消息（RFC 8446 Section 4.4.4）
