---
title: "AES-GCM"
category: "crypto"
type: ref
tags: [密码学, aes, gcm, aead, 认证加密]
created: 2026-05-15
updated: 2026-05-15
---

# AES-GCM

**类别**: 密码学

## 概述

AES-GCM（Galois/Counter Mode）是基于 AES 分组密码的认证加密模式，提供机密性和完整性保护。

## 原理

### AES 分组密码

AES（Advanced Encryption Standard）是分组密码，支持 128/192/256 位密钥。

**处理流程**：
1. 将明文分成 128 位（16 字节）的块
2. 对每个块进行加密
3. 输出密文块

### GCM 模式

GCM 结合了 CTR 模式加密和 GHASH 认证。

**加密过程**：
1. 生成初始计数器 J0
2. 使用 CTR 模式加密明文
3. 使用 GHASH 计算认证标签

**认证标签**：
```
T = GHASH(H, AAD || C || len(AAD) || len(C)) ⊕ E(J0)
```

- H: 哈希子密钥
- AAD: 附加认证数据
- C: 密文
- E(J0): 加密的初始计数器

### 安全性

- 128 位安全级别
- 支持附加认证数据（AAD）
- 抗时序攻击

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| aead_context | AEAD 加解密上下文 | [[crypto/aead|aead]] |
| encrypt_tls_record | Reality 加密 TLS 记录 | [[stealth/reality/response|response]] |
| seal | Reality 加密传输层 | [[stealth/reality/seal|seal]] |

## 参考资料

- [NIST SP 800-38D - Recommendation for GCM](https://csrc.nist.gov/publications/detail/sp/800-38d/final)

## 相关知识

- [[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]] — 替代的 AEAD 算法
- [[ref/protocol/tls-1.3|TLS 1.3]] — 使用 AES-128-GCM
