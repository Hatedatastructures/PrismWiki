---
title: "HMAC-SHA1"
category: "crypto"
type: ref
tags: [密码学, hmac, sha1, 消息认证码]
created: 2026-05-15
updated: 2026-05-15
---

# HMAC-SHA1

**类别**: 密码学

## 概述

HMAC-SHA1 是一种基于 SHA-1 哈希函数的消息认证码（MAC）算法，用于验证消息的完整性和真实性。

## 原理

### HMAC 结构

HMAC（Hash-based Message Authentication Code）使用加密哈希函数和密钥来生成消息认证码。

**算法步骤**：
1. 如果密钥长度大于哈希函数的块大小（SHA-1 为 64 字节），则对密钥进行哈希
2. 如果密钥长度小于块大小，则用零填充到块大小
3. 计算 inner_hash = H((K ⊕ ipad) || message)
4. 计算 HMAC = H((K ⊕ opad) || inner_hash)

其中：
- ipad = 0x36 重复 64 次
- opad = 0x5C 重复 64 次
- H 是 SHA-1 哈希函数

### SHA-1 哈希函数

SHA-1 生成 160 位（20 字节）的哈希值，处理 512 位（64 字节）的块。

**处理流程**：
1. 消息填充：添加 1 位，然后填充零，最后添加 64 位的消息长度
2. 初始化 5 个 32 位的寄存器（A, B, C, D, E）
3. 对每个 512 位块进行 80 轮处理
4. 输出 160 位的哈希值

**安全性**: SHA-1 已经被证明存在碰撞攻击，不建议用于安全敏感的应用。但在 HMAC 中使用 SHA-1 仍然是安全的，因为 HMAC 的安全性不依赖于哈希函数的抗碰撞性。

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| verify_client_hello | 用 HMAC-SHA1 验证 SessionID | [[stealth/shadowtls/auth|auth]] |
| compute_hmac | 计算 HMAC-SHA1 标签 | [[stealth/shadowtls/auth|auth]] |
| verify_frame_hmac | 验证数据帧 HMAC | [[stealth/shadowtls/auth|auth]] |
| compute_write_hmac | 计算写入 HMAC | [[stealth/shadowtls/auth|auth]] |

## 参考资料

- [RFC 2104 - HMAC: Keyed-Hashing for Message Authentication](https://tools.ietf.org/html/rfc2104)
- [RFC 3174 - US Secure Hash Algorithm 1 (SHA1)](https://tools.ietf.org/html/rfc3174)

## 相关知识

- [[ref/crypto/hmac-sha256|HMAC-SHA256]] — 更安全的 HMAC 变体
- [[ref/crypto/sha256|SHA-256]] — 更安全的哈希函数
- [[ref/protocol/tls-sessionid|TLS SessionID]] — ShadowTLS 如何使用 SessionID
