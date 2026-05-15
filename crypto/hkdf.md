---
title: "hkdf — HKDF-SHA256 密钥派生"
source: "include/prism/crypto/hkdf.hpp"
module: "crypto"
type: api
tags: [crypto, hkdf, 密钥派生, hmac, sha256, tls13]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[ref/crypto/hkdf|HKDF]]"
  - "[[ref/crypto/hmac-sha256|HMAC-SHA256]]"
  - "[[ref/crypto/sha256|SHA-256]]"
  - "[[ref/crypto/aead|AEAD]]"
  - "[[ref/stealth/reality|Reality]]"
  - "[[ref/fault/code|fault::code]]"
---

# hkdf.hpp

> 源码: `include/prism/crypto/hkdf.hpp`
> 实现: `src/prism/crypto/hkdf.cpp`
> 模块: [[ref/crypto|crypto]]

## 概述

HKDF-SHA256 密钥派生工具。提供 HMAC-SHA256、HMAC-SHA512、HKDF-Extract、HKDF-Expand 和 TLS 1.3 专用的 HKDF-Expand-Label 函数。这些函数是 TLS 1.3 密钥调度的核心组件，用于从 ECDHE 共享密钥派生握手和应用流量密钥。同时提供 SHA-256 哈希的多块重载。基于 BoringSSL 的 HMAC/SHA API 实现。

## 命名空间

`psm::crypto`

## 依赖关系

| 依赖 | 用途 | 层级 |
|------|------|------|
| `<prism/fault/code.hpp>` | 错误码 `fault::code` | fault |
| `<openssl/hmac.h>` | BoringSSL HMAC API | 外部 |
| `<openssl/sha.h>` | BoringSSL SHA256 API | 外部 |
| `<openssl/evp.h>` | BoringSSL EVP_MD API | 外部 |
| `<prism/trace.hpp>` | 错误日志 | trace |

## 常量

```cpp
constexpr std::size_t SHA256_LEN = 32;  // SHA-256 输出长度（字节）
constexpr std::size_t SHA512_LEN = 64;  // SHA-512 输出长度（字节）
```

## 函数

### `hmac_sha256()`

> 源码: include/prism/crypto/hkdf.hpp:42 | 实现: src/prism/crypto/hkdf.cpp:17

**功能**: 使用 HMAC-SHA256 算法计算消息认证码。

**签名**:
```cpp
[[nodiscard]] auto hmac_sha256(std::span<const std::uint8_t> key, std::span<const std::uint8_t> data)
    -> std::array<std::uint8_t, SHA256_LEN>;
```

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `key` | `std::span<const std::uint8_t>` | HMAC 密钥 |
| `data` | `std::span<const std::uint8_t>` | 待认证消息 |

**返回值**: `std::array<std::uint8_t, SHA256_LEN>` — 32 字节 HMAC-SHA256 结果。计算失败时返回全零。

**调用（向下）**: `HMAC(EVP_sha256())` (BoringSSL)

**被调用（向上）**: `hkdf_extract()`, `hkdf_expand()`, `reality::keygen` (Finished verify_data)

**知识域**: `[[ref/crypto/hmac-sha256|HMAC-SHA256]]`, `[[ref/crypto/sha256|SHA-256]]`

---

### `hmac_sha512()`

> 源码: include/prism/crypto/hkdf.hpp:51 | 实现: src/prism/crypto/hkdf.cpp:38

**功能**: 使用 HMAC-SHA512 算法计算消息认证码。

**签名**:
```cpp
[[nodiscard]] auto hmac_sha512(std::span<const std::uint8_t> key, std::span<const std::uint8_t> data)
    -> std::array<std::uint8_t, SHA512_LEN>;
```

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `key` | `std::span<const std::uint8_t>` | HMAC 密钥 |
| `data` | `std::span<const std::uint8_t>` | 待认证消息 |

**返回值**: `std::array<std::uint8_t, SHA512_LEN>` — 64 字节 HMAC-SHA512 结果。计算失败时返回全零。

**调用（向下）**: `HMAC(EVP_sha512())` (BoringSSL)

**被调用（向上）**: `reality::response` (Ed25519 自签名证书签名值计算)

**知识域**: `[[ref/crypto/hmac-sha256|HMAC-SHA256]]`, `[[ref/stealth/reality|Reality]]`

---

### `hkdf_extract()`

> 源码: include/prism/crypto/hkdf.hpp:62 | 实现: src/prism/crypto/hkdf.cpp:60

**功能**: HKDF-Extract 步骤，从输入密钥材料中提取伪随机密钥（PRK）。

**签名**:
```cpp
[[nodiscard]] auto hkdf_extract(std::span<const std::uint8_t> salt, std::span<const std::uint8_t> ikm)
    -> std::array<std::uint8_t, SHA256_LEN>;
```

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `salt` | `std::span<const std::uint8_t>` | 盐值（可以为空，为空时使用 32 字节全零） |
| `ikm` | `std::span<const std::uint8_t>` | 输入密钥材料 (Input Keying Material) |

**返回值**: `std::array<std::uint8_t, SHA256_LEN>` — 32 字节伪随机密钥 (PRK)。

**调用（向下）**: `hmac_sha256()`

**被调用（向上）**: `reality::keygen` (early_secret, handshake_secret, master_secret 提取)

**知识域**: `[[ref/crypto/hkdf|HKDF]]`, `[[ref/crypto/hmac-sha256|HMAC-SHA256]]`

---

### `hkdf_expand()`

> 源码: include/prism/crypto/hkdf.hpp:76 | 实现: src/prism/crypto/hkdf.cpp:80

**功能**: HKDF-Expand 步骤，将 PRK 扩展为指定长度的输出密钥材料。

**签名**:
```cpp
[[nodiscard]] auto hkdf_expand(std::span<const std::uint8_t> prk, std::span<const std::uint8_t> info,
                               std::size_t length)
    -> std::pair<fault::code, std::vector<std::uint8_t>>;
```

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `prk` | `std::span<const std::uint8_t>` | 伪随机密钥（至少 32 字节） |
| `info` | `std::span<const std::uint8_t>` | 上下文信息 |
| `length` | `std::size_t` | 输出长度（最大 255 x 32 = 8160 字节） |

**返回值**: `std::pair<fault::code, std::vector<std::uint8_t>>` — 错误码和输出字节的配对。成功时错误码为 `fault::code::success`。当 `length > 8160` 或 `prk.size() < 32` 时返回 `fault::code::invalid_argument`。

**调用（向下）**: `hmac_sha256()` (迭代调用)

**被调用（向上）**: `hkdf_expand_label()`, `reality::auth` (直接调用)

**知识域**: `[[ref/crypto/hkdf|HKDF]]`, `[[ref/crypto/hmac-sha256|HMAC-SHA256]]`, `[[ref/fault/code|fault::code]]`

---

### `hkdf_expand_label()`

> 源码: include/prism/crypto/hkdf.hpp:92 | 实现: src/prism/crypto/hkdf.cpp:150

**功能**: TLS 1.3 专用的密钥派生函数，按 RFC 8446 Section 7.1 构造 HkdfLabel 后调用 HKDF-Expand。

**签名**:
```cpp
[[nodiscard]] auto hkdf_expand_label(std::span<const std::uint8_t> secret, std::string_view label,
                                     std::span<const std::uint8_t> context, std::size_t length)
    -> std::pair<fault::code, std::vector<std::uint8_t>>;
```

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `secret` | `std::span<const std::uint8_t>` | 输入密钥 |
| `label` | `std::string_view` | 标签（如 `"key"`, `"iv"`, `"finished"`, `"c hs traffic"`） |
| `context` | `std::span<const std::uint8_t>` | 上下文数据（通常是 transcript hash） |
| `length` | `std::size_t` | 输出长度 |

**返回值**: `std::pair<fault::code, std::vector<std::uint8_t>>` — 错误码和输出字节的配对。TLS 1.3 自动在 label 前添加 `"tls13 "` 前缀。当 label 或 context 超过 255 字节时返回 `fault::code::invalid_argument`。

**调用（向下）**: `hkdf_expand()`

**被调用（向上）**: `reality::keygen` (所有 TLS 1.3 密钥派生：握手密钥、应用密钥、finished_key 等)

**知识域**: `[[ref/crypto/hkdf|HKDF]]`, `[[ref/fault/code|fault::code]]`

---

### `sha256()` — 单数据块

> 源码: include/prism/crypto/hkdf.hpp:102 | 实现: src/prism/crypto/hkdf.cpp:200

**功能**: 计算单个数据块的 SHA-256 哈希值。

**签名**:
```cpp
[[nodiscard]] auto sha256(std::span<const std::uint8_t> data)
    -> std::array<std::uint8_t, SHA256_LEN>;
```

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `std::span<const std::uint8_t>` | 输入数据 |

**返回值**: `std::array<std::uint8_t, SHA256_LEN>` — 32 字节 SHA-256 哈希值。

**调用（向下）**: `::SHA256()` (BoringSSL)

**被调用（向上）**: `reality::keygen` (计算空字符串哈希)

**知识域**: `[[ref/crypto/sha256|SHA-256]]`

---

### `sha256()` — 两数据块拼接

> 源码: include/prism/crypto/hkdf.hpp:113 | 实现: src/prism/crypto/hkdf.cpp:211

**功能**: 计算两个数据块拼接后的 SHA-256 哈希值，等价于 SHA-256(data1 || data2)。

**签名**:
```cpp
[[nodiscard]] auto sha256(std::span<const std::uint8_t> data1, std::span<const std::uint8_t> data2)
    -> std::array<std::uint8_t, SHA256_LEN>;
```

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `data1` | `std::span<const std::uint8_t>` | 第一个数据块 |
| `data2` | `std::span<const std::uint8_t>` | 第二个数据块 |

**返回值**: `std::array<std::uint8_t, SHA256_LEN>` — 32 字节 SHA-256 哈希值。

**调用（向下）**: `EVP_DigestInit_ex`, `EVP_DigestUpdate`, `EVP_DigestFinal_ex` (BoringSSL EVP 流式 API)

**被调用（向上）**: `reality::keygen` (hello_hash), `reality::response` (cv_transcript), `reality::handshake` (transcript_for_finished)

**知识域**: `[[ref/crypto/sha256|SHA-256]]`

---

### `sha256()` — 三数据块拼接

> 源码: include/prism/crypto/hkdf.hpp:124 | 实现: src/prism/crypto/hkdf.cpp:245

**功能**: 计算三个数据块拼接后的 SHA-256 哈希值，等价于 SHA-256(data1 || data2 || data3)。

**签名**:
```cpp
[[nodiscard]] auto sha256(std::span<const std::uint8_t> data1, std::span<const std::uint8_t> data2,
                          std::span<const std::uint8_t> data3)
    -> std::array<std::uint8_t, SHA256_LEN>;
```

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `data1` | `std::span<const std::uint8_t>` | 第一个数据块 |
| `data2` | `std::span<const std::uint8_t>` | 第二个数据块 |
| `data3` | `std::span<const std::uint8_t>` | 第三个数据块 |

**返回值**: `std::array<std::uint8_t, SHA256_LEN>` — 32 字节 SHA-256 哈希值。

**调用（向下）**: `EVP_DigestInit_ex`, `EVP_DigestUpdate`, `EVP_DigestFinal_ex` (BoringSSL EVP 流式 API)

**被调用（向上）**: `reality::response` (transcript_for_finished), `reality::handshake` (full_transcript_hash)

**知识域**: `[[ref/crypto/sha256|SHA-256]]`, `[[ref/stealth/reality|Reality]]`
