---
title: "DNS over UDP"
category: "protocol"
type: ref
tags: [协议, dns, udp]
created: 2026-05-15
updated: 2026-05-15
---

# DNS over UDP

**类别**: 协议

## 概述

DNS over UDP 是传统的 DNS 查询方式，使用 UDP 协议进行 DNS 解析。端口 53，单个查询/响应。

## 原理

### 报文格式

```
DNS Message:
  Header: 12 bytes
    ID: 2 bytes
    Flags: 2 bytes
    QDCOUNT: 2 bytes
    ANCOUNT: 2 bytes
    NSCOUNT: 2 bytes
    ARCOUNT: 2 bytes
  Question: variable
  Answer: variable
  Authority: variable
  Additional: variable
```

### 限制

- UDP 报文最大 512 字节（传统）
- EDNS0 扩展支持更大报文（最大 4096 字节）
- 不支持加密

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| udp_query | UDP DNS 查询 | [[resolve/dns/upstream|upstream]] |

## 参考资料

- [RFC 1035 - Domain Names](https://tools.ietf.org/html/rfc1035)

## 相关知识

- [[ref/protocol/dns-over-tcp|DNS over TCP]] — TCP DNS 查询
- [[ref/protocol/dns-over-tls|DNS over TLS]] — DoT
- [[ref/protocol/dns-over-https|DNS over HTTPS]] — DoH
