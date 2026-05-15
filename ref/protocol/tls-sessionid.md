---
title: "TLS SessionID"
category: "protocol"
type: ref
tags: [协议, tls, sessionid, 伪装]
created: 2026-05-15
updated: 2026-05-15
---

# TLS SessionID

**类别**: 协议

## 概述

TLS SessionID 是 ClientHello 中的变长字段（0-32 字节），用于会话恢复。ShadowTLS 利用这个字段嵌入 4 字节 HMAC 标签进行身份认证。

## 原理

### 字段结构

```
SessionID:
  Length: 1 byte (0-32)
  Data: 0-32 bytes
```

### ShadowTLS 的利用方式

ShadowTLS v3 要求 SessionID 固定 32 字节，在最后 4 字节嵌入 HMAC-SHA1 标签：

```
SessionID[0:28] = 随机数据
SessionID[28:32] = HMAC-SHA1(password, ClientHello[10:28] + 00000000 + ClientHello[32:])[:4]
```

### Reality 的利用方式

Reality 在 SessionID 前 3 字节嵌入独占标记：
```
SessionID[0:3] = [0x01, 0x08, 0x02]  (Reality 标记)
```

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| verify_client_hello | 验证 SessionID HMAC | [[stealth/shadowtls/auth|auth]] |
| sniff | 检查 SessionID 标记 | [[stealth/reality/scheme|scheme]] |
| parse_client_hello | 提取 SessionID | [[stealth/reality/request|request]] |

## 参考资料

- [RFC 8446 Section 4.1.2 - ClientHello](https://tools.ietf.org/html/rfc8446#section-4.1.2)

## 相关知识

- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 结构
- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — ShadowTLS 使用的 HMAC
