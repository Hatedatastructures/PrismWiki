---
title: "DNS over TLS"
category: "protocol"
type: ref
tags: [协议, dns, tls, dot]
created: 2026-05-15
updated: 2026-05-15
---

# DNS over TLS (DoT)

**类别**: 协议

## 概述

DNS over TLS (DoT) 使用 TLS 加密 DNS 查询，端口 853。提供隐私保护，防止 DNS 查询被窃听或篡改。

## 原理

### 连接流程

```
Client                    DNS Server
  |                          |
  |--- TLS 握手 ----------->|
  |<-- TLS 握手 ------------|
  |                          |
  |--- DNS 查询 (加密) ---->|
  |<-- DNS 响应 (加密) -----|
```

### 优势

- 加密 DNS 查询
- 防止 DNS 劫持
- 端口固定（853），容易识别

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| dot_query | DoT 查询 | [[resolve/dns/upstream|upstream]] |

## 参考资料

- [RFC 7858 - Specification for DNS over TLS](https://tools.ietf.org/html/rfc7858)

## 相关知识

- [[ref/protocol/dns-over-udp|DNS over UDP]] — UDP DNS 查询
- [[ref/protocol/dns-over-https|DNS over HTTPS]] — DoH
