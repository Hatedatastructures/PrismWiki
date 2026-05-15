---
title: "ChaCha20-Poly1305"
category: "crypto"
type: ref
tags: [密码学, chacha20, poly1305, aead, 认证加密]
created: 2026-05-15
updated: 2026-05-15
---

# ChaCha20-Poly1305

**类别**: 密码学

## 概述

ChaCha20-Poly1305 是基于 ChaCha20 流密码和 Poly1305 消息认证码的认证加密算法。

## 原理

### ChaCha20 流密码

ChaCha20 是 Salsa20 的改进版本，生成伪随机密钥流。

**处理流程**：
1. 初始化 4×4 矩阵（常量、密钥、计数器、Nonce）
2. 进行 20 轮列轮和对角轮操作
3. 输出 64 字节密钥流
4. 密钥流与明文异或得到密文

### Poly1305 消息认证码

Poly1305 是快速的消息认证码，生成 16 字节认证标签。

**处理流程**：
1. 将消息分成 16 字节块
2. 对每个块进行多项式计算
3. 输出 16 字节认证标签

### AEAD 构造

```
C = ChaCha20(key, nonce, plaintext)
T = Poly1305(key, AAD || C || len(AAD) || len(C))
```

### 安全性

- 128 位安全级别
- 抗时序攻击
- 适合软件实现

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| aead_context | AEAD 加解密上下文 | [[crypto/aead|aead]] |

## 参考资料

- [RFC 8439 - ChaCha20 and Poly1305 for IETF Protocols](https://tools.ietf.org/html/rfc8439)

## 相关知识

- [[ref/crypto/aes-gcm|AES-GCM]] — 替代的 AEAD 算法
- [[ref/protocol/tls-1.3|TLS 1.3]] — 支持 ChaCha20-Poly1305
