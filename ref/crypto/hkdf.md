---
title: "HKDF"
category: "crypto"
type: ref
tags: [密码学, hkdf, 密钥派生]
created: 2026-05-15
updated: 2026-05-15
---

# HKDF

**类别**: 密码学

## 概述

HKDF（HMAC-based Key Derivation Function）是基于 HMAC 的密钥派生函数，用于从输入密钥材料派生出一个或多个加密密钥。

## 原理

### HKDF 结构

HKDF 分为两个阶段：
1. **提取（Extract）**: 从输入密钥材料中提取伪随机密钥
2. **扩展（Expand）**: 从伪随机密钥派生出所需长度的输出密钥

### 提取阶段

```
PRK = HMAC-Hash(salt, IKM)
```

- PRK: 伪随机密钥（与 Hash 函数输出长度相同）
- salt: 盐值（可选）
- IKM: 输入密钥材料

### 扩展阶段

```
T(0) = empty
T(i) = HMAC-Hash(PRK, T(i-1) || info || i)
OKM = T(1) || T(2) || ... || T(N)
```

- OKM: 输出密钥材料
- info: 上下文信息
- i: 计数器（1 字节）

### 安全性

- 基于 HMAC 的安全性
- 可以从弱密钥材料派生出强密钥
- 支持任意长度的输出

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| hkdf_expand_label | TLS 1.3 密钥派生 | [[crypto/hkdf|hkdf]] |
| derive_handshake_keys | Reality 握手密钥派生 | [[stealth/reality/keygen|keygen]] |
| derive_application_keys | Reality 应用密钥派生 | [[stealth/reality/keygen|keygen]] |

## 参考资料

- [RFC 5869 - HMAC-based Extract-and-Expand Key Derivation Function (HKDF)](https://tools.ietf.org/html/rfc5869)

## 相关知识

- [[ref/crypto/hmac-sha256|HMAC-SHA256]] — HKDF 使用的 HMAC
- [[ref/crypto/x25519|X25519]] — 密钥交换后使用 HKDF 派生密钥
- [[ref/protocol/tls-1.3|TLS 1.3]] — 使用 HKDF 进行密钥派生
