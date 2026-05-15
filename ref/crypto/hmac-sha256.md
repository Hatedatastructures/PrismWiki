---
title: "HMAC-SHA256"
category: "crypto"
type: ref
tags: [密码学, hmac, sha256, 消息认证码]
created: 2026-05-15
updated: 2026-05-15
---

# HMAC-SHA256

**类别**: 密码学

## 概述

HMAC-SHA256 是一种基于 SHA-256 哈希函数的消息认证码算法，提供更强的安全性。

## 原理

### HMAC 结构

HMAC-SHA256 使用 SHA-256 作为底层哈希函数，生成 256 位（32 字节）的消息认证码。

**算法步骤**：
1. 如果密钥长度大于 64 字节，则对密钥进行 SHA-256 哈希
2. 如果密钥长度小于 64 字节，则用零填充到 64 字节
3. 计算 inner_hash = SHA256((K ⊕ ipad) || message)
4. 计算 HMAC = SHA256((K ⊕ opad) || inner_hash)

### SHA-256 哈希函数

SHA-256 是 SHA-2 家族的一部分，生成 256 位的哈希值。

**处理流程**：
1. 消息填充：添加 1 位，然后填充零，最后添加 64 位的消息长度
2. 初始化 8 个 32 位的寄存器（A, B, C, D, E, F, G, H）
3. 对每个 512 位块进行 64 轮处理
4. 输出 256 位的哈希值

**安全性**: SHA-256 目前被认为是安全的，没有已知的实际攻击。

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| compute_write_key | 使用 SHA-256 生成写入密钥 | [[stealth/shadowtls/auth|auth]] |
| hkdf_expand_label | HKDF 使用 HMAC-SHA256 | [[crypto/hkdf|hkdf]] |
| compute_finished_verify_data | 使用 HMAC-SHA256 计算 Finished | [[stealth/reality/keygen|keygen]] |

## 参考资料

- [RFC 2104 - HMAC: Keyed-Hashing for Message Authentication](https://tools.ietf.org/html/rfc2104)
- [RFC 6234 - US Secure Hash Algorithms](https://tools.ietf.org/html/rfc6234)

## 相关知识

- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — 较弱的 HMAC 变体
- [[ref/crypto/sha256|SHA-256]] — 底层哈希函数
- [[ref/crypto/hkdf|HKDF]] — 使用 HMAC-SHA256 的密钥派生
