---
layer: core
source:
  - I:/code/Prism/include/prism/crypto/aead.hpp
  - I:/code/Prism/include/prism/crypto/hkdf.hpp
  - I:/code/Prism/include/prism/crypto/x25519.hpp
  - I:/code/Prism/include/prism/crypto/blake3.hpp
  - I:/code/Prism/include/prism/crypto/block.hpp
  - I:/code/Prism/include/prism/crypto/base64.hpp
title: Crypto 加密模块
---

# Crypto 加密模块

Prism 加密模块提供类型安全的密码学原语封装，基于 BoringSSL 实现，支持现代加密协议所需的全部核心功能。

## 模块组成

| 模块 | 功能 | 应用场景 |
|------|------|----------|
| [[core/crypto/aead|aead]] | AEAD 认证加密 | TLS 1.3、SS2022 数据加密 |
| [[core/crypto/hkdf|hkdf]] | HKDF 密钥派生 | TLS 1.3 密钥调度 |
| [[core/crypto/x25519|x25519]] | X25519 密钥交换 | Reality 协议 ECDHE |
| [[core/crypto/blake3|blake3]] | BLAKE3 密钥派生 | SS2022 会话密钥派生 |
| [[core/crypto/block|block]] | AES-ECB 单块加密 | SS2022 UDP SeparateHeader |
| [[core/crypto/base64|base64]] | Base64 编解码 | HTTP Basic 认证 |

## 设计原则

### 类型安全

所有函数使用 `std::span` 和 `std::array` 传递密钥和数据，避免裸指针和缓冲区溢出：

```cpp
// 固定长度密钥
std::array<std::uint8_t, 32> key;
auto ctx = aead_context(aead_cipher::aes_256_gcm, key);

// 动态长度数据
std::vector<std::uint8_t> plaintext;
std::vector<std::uint8_t> ciphertext(plaintext.size() + aead_context::tag_length());
ctx.seal(ciphertext, plaintext);
```

### 错误处理

加密操作返回 `fault::code` 枚举，区分成功与各类错误：

```cpp
auto result = ctx.seal(out, plaintext, ad);
if (result != fault::code::success) {
    // 处理加密错误
}
```

### 资源管理

使用 RAII 管理加密上下文生命周期，支持移动语义：

```cpp
aead_context ctx1(aead_cipher::aes_256_gcm, key);
aead_context ctx2 = std::move(ctx1);  // 移动构造
// ctx1 置为无效状态，ctx2 接管资源
```

## 加密算法支持

### AEAD 算法

| 算法 | 密钥长度 | Nonce 长度 | Tag 长度 | 用途 |
|------|----------|------------|----------|------|
| AES-128-GCM | 16 字节 | 12 字节 | 16 字节 | TLS 1.3 |
| AES-256-GCM | 32 字节 | 12 字节 | 16 字节 | TLS 1.3、SS2022 |
| ChaCha20-Poly1305 | 32 字节 | 12 字节 | 16 字节 | TLS 1.3、SS2022 |
| XChaCha20-Poly1305 | 32 字节 | 24 字节 | 16 字节 | SS2022 |

### 密钥派生

| 算法 | 输出长度 | 用途 |
|------|----------|------|
| HKDF-SHA256 | 可变（最大 8160 字节） | TLS 1.3 密钥调度 |
| BLAKE3 derive_key | 可变 | SS2022 会话子密钥 |

### 密钥交换

| 算法 | 密钥长度 | 共享密钥长度 | 安全强度 |
|------|----------|--------------|----------|
| X25519 | 32 字节 | 32 字节 | 128 位 |

## 依赖关系

```
┌─────────────────────────────────────────────────────────┐
│                     Protocol Layer                       │
│  TLS 1.3 │ SS2022 │ Reality │ Trojan │ VLESS │ SOCKS5  │
└────┬──────────┬─────────┬──────────┬──────────────┬─────┘
     │          │         │          │              │
     ▼          ▼         ▼          ▼              ▼
┌─────────────────────────────────────────────────────────┐
│                     Crypto Layer                         │
│  ┌──────┐ ┌──────┐ ┌────────┐ ┌────────┐ ┌───────────┐  │
│  │ AEAD │ │ HKDF │ │ X25519 │ │ BLAKE3 │ │ AES Block │  │
│  └──┬───┘ └──┬───┘ └───┬────┘ └───┬────┘ └─────┬─────┘  │
│     │        │         │          │            │        │
│     ▼        ▼         ▼          ▼            ▼        │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   BoringSSL / OpenSSL                    │
└─────────────────────────────────────────────────────────┘
```

## 相关文档

- [[core/crypto/aead|aead]] - AEAD 认证加密详解
- [[core/crypto/hkdf|hkdf]] - HKDF 密钥派生详解
- [[core/crypto/x25519|x25519]] - X25519 密钥交换详解
- [[core/crypto/blake3|blake3]] - BLAKE3 密钥派生详解
- [[core/crypto/block|block]] - AES-ECB 单块加密详解
- [[core/crypto/base64|base64]] - Base64 编解码详解