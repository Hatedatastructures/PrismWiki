---
title: "TLS ServerHello"
category: "protocol"
type: ref
tags: [协议, tls, serverhello, 握手]
created: 2026-05-15
updated: 2026-05-15
---

# TLS ServerHello

**类别**: 协议

## 概述

ServerHello 是 TLS 握手的第二条消息，服务器发送给客户端，包含选定的协议版本、密码套件、扩展等信息。

## 原理

### 报文格式

```
ServerHello:
  ProtocolVersion: 2 bytes
  Random: 32 bytes (server random)
  SessionID: 0-32 bytes (echo)
  CipherSuite: 2 bytes
  CompressionMethod: 1 byte
  Extensions: variable
```

### ServerRandom

ServerRandom 是 32 字节的随机数，用于密钥派生。ShadowTLS 利用 ServerRandom 生成写入密钥。

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| generate_server_hello | 生成 ServerHello | [[stealth/reality/response|response]] |
| derive_handshake_keys | 使用 ServerRandom 派生密钥 | [[stealth/reality/keygen|keygen]] |
| compute_write_key | 使用 ServerRandom 生成写入密钥 | [[stealth/shadowtls/auth|auth]] |

## 参考资料

- [RFC 8446 Section 4.1.3 - ServerHello](https://tools.ietf.org/html/rfc8446#section-4.1.3)

## 相关知识

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 协议
- [[ref/protocol/tls-serverrandom|TLS ServerRandom]] — ServerRandom 字段详解
