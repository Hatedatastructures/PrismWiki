---
title: "TLS ClientHello"
category: "protocol"
type: ref
tags: [协议, tls, clienthello, 握手]
created: 2026-05-15
updated: 2026-05-15
---

# TLS ClientHello

**类别**: 协议

## 概述

ClientHello 是 TLS 握手的第一条消息，客户端发送给服务器，包含支持的协议版本、密码套件、扩展等信息。Reality 和 ShadowTLS 通过分析 ClientHello 进行伪装检测。

## 原理

### 报文格式

```
ClientHello:
  ProtocolVersion: 2 bytes (legacy_version)
  Random: 32 bytes (client random)
  SessionID: 0-32 bytes (legacy_session_id)
  CipherSuites: 2 + N*2 bytes
  CompressionMethods: 1 + N bytes
  Extensions: variable
```

### 关键扩展

| 扩展 | 类型 | 说明 |
|------|------|------|
| server_name (SNI) | 0x0000 | 服务器名称 |
| supported_versions | 0x002B | 支持的 TLS 版本 |
| key_share | 0x0033 | 密钥交换公钥 |
| signature_algorithms | 0x000D | 支持的签名算法 |

### SessionID 字段

SessionID 是 0-32 字节的变长字段。Reality 和 ShadowTLS 利用这个字段嵌入认证信息：
- **Reality**: session_id[0:3] = [0x01, 0x08, 0x02]（独占标记）
- **ShadowTLS**: session_id 后 4 字节嵌入 HMAC-SHA1 标签

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| parse_client_hello | 解析 ClientHello | [[stealth/reality/request|request]] |
| verify_client_hello | 验证 SessionID HMAC | [[stealth/shadowtls/auth|auth]] |
| sniff | Tier 0 快速检测 | [[stealth/scheme|scheme]] |
| analyze | 特征分析 | [[recognition/probe/analyzer|analyzer]] |

## 参考资料

- [RFC 8446 Section 4.1.2 - ClientHello](https://tools.ietf.org/html/rfc8446#section-4.1.2)

## 相关知识

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 协议
- [[ref/protocol/tls-sessionid|TLS SessionID]] — SessionID 字段详解
- [[ref/protocol/tls-extensions|TLS 扩展]] — TLS 扩展详解
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — JA3/JA4 指纹
