---
title: "TLS ServerRandom"
category: "protocol"
type: ref
tags: [协议, tls, serverrandom, 密钥]
created: 2026-05-15
updated: 2026-05-15
---

# TLS ServerRandom

**类别**: 协议

## 概述

TLS ServerRandom 是 ServerHello 中的 32 字节随机数，用于密钥派生。ShadowTLS 利用 ServerRandom 生成写入密钥和计算数据帧 HMAC。

## 原理

### 字段结构

```
ServerRandom: 32 bytes (固定长度)
```

### 在密钥派生中的作用

ServerRandom 参与 TLS 1.3 密钥派生：
```
shared_secret = ECDHE(client_private, server_public)
handshake_secret = HKDF-Extract(shared_secret, early_secret)
```

### ShadowTLS 的利用方式

ShadowTLS 使用 ServerRandom 生成写入密钥和计算数据帧 HMAC：
```
WriteKey = SHA256(password + serverRandom)
HMAC_Verify = HMAC-SHA1(password, serverRandom + "C" + payload)[:4]
HMAC_Write = HMAC-SHA1(password, serverRandom + payload)[:4]
```

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| compute_write_key | 使用 ServerRandom 生成写入密钥 | [[stealth/shadowtls/auth|auth]] |
| verify_frame_hmac | 使用 ServerRandom 验证数据帧 | [[stealth/shadowtls/auth|auth]] |
| compute_write_hmac | 使用 ServerRandom 计算写入 HMAC | [[stealth/shadowtls/auth|auth]] |
| derive_handshake_keys | 使用 ServerRandom 派生密钥 | [[stealth/reality/keygen|keygen]] |

## 参考资料

- [RFC 8446 Section 4.1.3 - ServerHello](https://tools.ietf.org/html/rfc8446#section-4.1.3)

## 相关知识

- [[ref/protocol/tls-serverhello|TLS ServerHello]] — ServerHello 结构
- [[ref/protocol/tls-sessionid|TLS SessionID]] — SessionID 字段
- [[ref/crypto/sha256|SHA-256]] — ShadowTLS 使用的哈希
