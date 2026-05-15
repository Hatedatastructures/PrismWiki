---
title: "DNS over HTTPS"
category: "protocol"
type: ref
tags: [协议, dns, https, doh]
created: 2026-05-15
updated: 2026-05-15
---

# DNS over HTTPS (DoH)

**类别**: 协议

## 概述

DNS over HTTPS (DoH) 使用 HTTPS 协议进行 DNS 查询，与普通 HTTPS 流量混合，难以被识别和封锁。

## 原理

### 请求格式

```
GET /dns-query?dns=base64_dns_message HTTP/1.1
Host: dns.example.com
Accept: application/dns-message
```

### 优势

- 与 HTTPS 流量混合，难以识别
- 支持 HTTP/2 多路复用
- 可以使用 CDN

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| doh_query | DoH 查询 | [[resolve/dns/upstream|upstream]] |

## 参考资料

- [RFC 8484 - DNS Queries over HTTPS](https://tools.ietf.org/html/rfc8484)

## 相关知识

- [[ref/protocol/dns-over-tls|DNS over TLS]] — DoT
- [[ref/protocol/dns-over-udp|DNS over UDP]] — UDP DNS 查询
