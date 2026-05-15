---
title: "TLS 扩展"
category: "protocol"
type: ref
tags: [协议, tls, extensions, 扩展]
created: 2026-05-15
updated: 2026-05-15
---

# TLS 扩展

**类别**: 协议

## 概述

TLS 扩展是 ClientHello/ServerHello 中的可选字段，用于协商额外功能。Prism 的伪装方案通过分析扩展进行检测。

## 原理

### 扩展格式

```
Extension:
  ExtensionType: 2 bytes
  ExtensionData: variable
```

### 关键扩展

| 扩展 | 类型 | 说明 |
|------|------|------|
| server_name (SNI) | 0x0000 | 服务器名称，用于路由 |
| supported_versions | 0x002B | 支持的 TLS 版本 |
| key_share | 0x0033 | 密钥交换公钥 |
| signature_algorithms | 0x000D | 支持的签名算法 |
| application_layer_protocol_negotiation (ALPN) | 0x0010 | 应用层协议协商 |
| encrypted_client_hello (ECH) | 0xFE0D | 加密 ClientHello |

### SNI 扩展

SNI（Server Name Indication）扩展携带服务器名称，Prism 用于：
- 路由到不同的伪装方案
- Reality 检查 SNI 是否在白名单中

### key_share 扩展

key_share 扩展携带密钥交换公钥，Prism 用于：
- 提取 X25519 公钥进行密钥交换
- 识别支持的密钥交换算法

### ECH 扩展

ECH（Encrypted Client Hello）扩展加密 ClientHello 中的敏感信息（如 SNI），Prism 支持解密 ECH。

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| parse_client_hello | 提取扩展 | [[stealth/reality/request|request]] |
| sniff | 分析扩展特征 | [[stealth/scheme|scheme]] |
| match_server_name | 检查 SNI | [[stealth/reality/auth|auth]] |
| decrypt_ech_payload | 解密 ECH | [[stealth/ech/decrypt|decrypt]] |

## 参考资料

- [RFC 8446 Section 4.2 - Extensions](https://tools.ietf.org/html/rfc8446#section-4.2)

## 相关知识

- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 结构
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 协议
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — JA3/JA4 指纹
