---
title: "HTTP CONNECT 代理"
category: "protocol"
type: ref
tags: [协议, http, connect, 代理]
created: 2026-05-15
updated: 2026-05-15
---

# HTTP CONNECT 代理

**类别**: 协议

## 概述

HTTP CONNECT 是 HTTP 代理的一种方法，通过建立隧道连接来代理 TCP 流量。客户端发送 CONNECT 请求，服务器建立隧道后透传数据。

## 原理

### 连接流程

```
Client                    Proxy                   Target
  |                         |                        |
  |--- CONNECT host:port -->|                        |
  |<-- 200 Connection ----->|                        |
  |                         |                        |
  |<========= 隧道 ===============>|
```

### 请求格式

```
CONNECT host:port HTTP/1.1
Host: host:port
Proxy-Authorization: Basic base64(user:pass)  (可选)
```

### 响应格式

```
HTTP/1.1 200 Connection Established
```

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| http relay | HTTP 代理 | [[protocol/http/relay|relay]] |
| http parser | HTTP 解析 | [[protocol/http/parser|parser]] |

## 参考资料

- [RFC 7231 Section 4.3.6 - CONNECT](https://tools.ietf.org/html/rfc7231#section-4.3.6)

## 相关知识

- [[ref/protocol/socks5-rfc1928|SOCKS5]] — SOCKS5 代理
