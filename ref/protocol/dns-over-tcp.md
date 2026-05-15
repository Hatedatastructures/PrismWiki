---
title: "DNS over TCP"
category: "protocol"
type: ref
tags: [协议, dns, tcp]
created: 2026-05-15
updated: 2026-05-15
---

# DNS over TCP

**类别**: 协议

## 概述

DNS over TCP 使用 TCP 协议进行 DNS 解析，支持更大的报文和更可靠的传输。

## 原理

### 报文格式

```
TCP DNS:
  Length: 2 bytes (报文长度)
  DNS Message: variable
```

### 优势

- 支持大于 512 字节的报文
- 可靠传输，无丢包
- 支持 DNSSEC

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| tcp_query | TCP DNS 查询 | [[resolve/dns/upstream|upstream]] |

## 参考资料

- [RFC 7766 - DNS Transport over TCP](https://tools.ietf.org/html/rfc7766)

## 相关知识

- [[ref/protocol/dns-over-udp|DNS over UDP]] — UDP DNS 查询
- [[ref/protocol/dns-over-tls|DNS over TLS]] — DoT
