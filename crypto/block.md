---
title: "block — AES-ECB 单块加解密"
source: "include/prism/crypto/block.hpp"
module: "crypto"
type: api
tags: [crypto, block, aes, ecb, 加密]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[ref/crypto/aes-gcm|AES-GCM]]"
  - "[[crypto/aead|AEAD]]"
  - "[[pipeline/protocols/shadowsocks|Shadowsocks]]"
---

# block.hpp

> 源码: `include/prism/crypto/block.hpp`
> 实现: `src/prism/crypto/block.cpp`
> 模块: [[crypto|crypto]]

## 概述

AES-ECB 单块加解密工具。提供 AES-ECB 单块（16 字节）加密和解密功能，用于 SS2022 (SIP022) UDP 的 SeparateHeader 加密。不应直接用于大量数据加密（ECB 模式不安全）。

## 命名空间

`psm::crypto`

## 依赖关系

| 依赖 | 用途 | 层级 |
|------|------|------|
| `<openssl/evp.h>` | BoringSSL EVP_CIPHER API | 外部 |
| `<array>` | `std::array` 返回值 | 标准库 |
| `<span>` | `std::span` 输入参数 | 标准库 |

## 函数

### aes_ecb_encrypt

功能说明：对单个 16 字节明文块执行 AES-ECB 加密，返回 16 字节密文。

签名：

```cpp
[[nodiscard]] auto aes_ecb_encrypt(std::span<const std::uint8_t, 16> input,
                                   std::span<const std::uint8_t> key)
    -> std::array<std::uint8_t, 16>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `input` | `std::span<const std::uint8_t, 16>` | 明文块，固定 16 字节 |
| `key` | `std::span<const std::uint8_t>` | AES 密钥，16 字节（AES-128）或 32 字节（AES-256） |

**返回值**：16 字节密文。若 EVP 上下文初始化或加密操作失败则返回全零数组。

**调用（向下）**：
- `EVP_CIPHER_CTX_new()` — 创建 BoringSSL 密码上下文
- `EVP_aes_128_ecb()` / `EVP_aes_256_ecb()` — 根据密钥长度选择算法
- `EVP_EncryptInit_ex()` — 初始化加密上下文
- `EVP_CIPHER_CTX_set_padding()` — 禁用填充（输入已是完整块）
- `EVP_EncryptUpdate()` — 执行加密
- `EVP_EncryptFinal_ex()` — 完成加密
- `EVP_CIPHER_CTX_free()` — 释放上下文

**被调用（向上）**：
- `protocol::shadowsocks::datagram` — 加密 SS2022 UDP SeparateHeader

**涉及的知识域**：
- [[ref/crypto/aes-gcm|AES-GCM]] — AES 算法基础与分组密码原理
- [[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]] — SS2022 的另一种 AEAD 算法

---

### aes_ecb_decrypt

功能说明：对单个 16 字节密文块执行 AES-ECB 解密，返回 16 字节明文。

签名：

```cpp
[[nodiscard]] auto aes_ecb_decrypt(std::span<const std::uint8_t, 16> input,
                                   std::span<const std::uint8_t> key)
    -> std::array<std::uint8_t, 16>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `input` | `std::span<const std::uint8_t, 16>` | 密文块，固定 16 字节 |
| `key` | `std::span<const std::uint8_t>` | AES 密钥，16 字节（AES-128）或 32 字节（AES-256） |

**返回值**：16 字节明文。若 EVP 上下文初始化或解密操作失败则返回全零数组。

**调用（向下）**：
- `EVP_CIPHER_CTX_new()` — 创建 BoringSSL 密码上下文
- `EVP_aes_128_ecb()` / `EVP_aes_256_ecb()` — 根据密钥长度选择算法
- `EVP_DecryptInit_ex()` — 初始化解密上下文
- `EVP_CIPHER_CTX_set_padding()` — 禁用填充
- `EVP_DecryptUpdate()` — 执行解密
- `EVP_DecryptFinal_ex()` — 完成解密
- `EVP_CIPHER_CTX_free()` — 释放上下文

**被调用（向上）**：
- `protocol::shadowsocks::datagram` — 解密 SS2022 UDP SeparateHeader

**涉及的知识域**：
- [[ref/crypto/aes-gcm|AES-GCM]] — AES 算法基础与分组密码原理
- [[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]] — SS2022 的另一种 AEAD 算法
