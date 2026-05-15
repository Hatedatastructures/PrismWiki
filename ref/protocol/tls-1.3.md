---
title: "TLS 1.3"
category: "protocol"
type: ref
tags: [协议, tls, 握手, 加密]
created: 2026-05-15
updated: 2026-05-15
---

# TLS 1.3

**类别**: 协议

## 概述

TLS 1.3 是最新版本的传输层安全协议，提供加密通信。相比 TLS 1.2，TLS 1.3 简化了握手流程（1-RTT），移除了不安全的密码套件，强制前向保密。

## 原理

### 1-RTT 握手流程

```
Client                              Server
  |                                    |
  |------- ClientHello -------------->|
  |       (supported_versions,        |
  |        key_share,                 |
  |        signature_algorithms)      |
  |                                    |
  |<------ ServerHello ---------------|
  |       (key_share,                 |
  |        cipher_suite)              |
  |<------ EncryptedExtensions -------|
  |<------ Certificate ---------------|
  |<------ CertificateVerify ---------|
  |<------ Finished ------------------|
  |                                    |
  |------- Finished ----------------->|
  |                                    |
  |<==== Application Data ===========>|
```

### 关键变化（相比 TLS 1.2）

- **握手从 2-RTT 减少到 1-RTT**
- **移除 RSA 密钥交换**（强制前向保密）
- **移除 CBC 模式密码套件**
- **移除压缩、重协商等不安全特性**
- **加密握手消息**（Certificate、CertificateVerify 等）

### 密码套件

TLS 1.3 仅支持 5 个密码套件：
- `TLS_AES_128_GCM_SHA256` (0x1301)
- `TLS_AES_256_GCM_SHA384` (0x1302)
- `TLS_CHACHA20_POLY1305_SHA256` (0x1303)
- `TLS_AES_128_CCM_SHA256` (0x1304)
- `TLS_AES_128_CCM_8_SHA256` (0x1305)

### 密钥派生

TLS 1.3 使用 HKDF 进行密钥派生：
```
Early Secret = HKDF-Extract(0, PSK)
Handshake Secret = HKDF-Extract(ECDHE, Early Secret)
Master Secret = HKDF-Extract(0, Handshake Secret)
```

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| handshake | Reality TLS 1.3 握手 | [[stealth/reality/handshake|handshake]] |
| derive_handshake_keys | TLS 1.3 密钥派生 | [[stealth/reality/keygen|keygen]] |
| generate_server_hello | 生成 ServerHello | [[stealth/reality/response|response]] |
| encrypt_tls_record | 加密 TLS 记录 | [[stealth/reality/seal|seal]] |
| parse_client_hello | 解析 ClientHello | [[stealth/reality/request|request]] |

## 参考资料

- [RFC 8446 - The Transport Layer Security (TLS) Protocol Version 1.3](https://tools.ietf.org/html/rfc8446)

## 相关知识

- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 结构
- [[ref/protocol/tls-serverhello|TLS ServerHello]] — ServerHello 结构
- [[ref/protocol/tls-extensions|TLS 扩展]] — TLS 扩展
- [[ref/crypto/hkdf|HKDF]] — TLS 1.3 密钥派生
- [[ref/crypto/aes-gcm|AES-GCM]] — TLS 1.3 密码套件
