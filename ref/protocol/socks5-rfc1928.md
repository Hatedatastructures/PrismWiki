---
title: "SOCKS5 协议"
category: "protocol"
type: ref
tags: [协议, socks5, 代理]
created: 2026-05-15
updated: 2026-05-15
---

# SOCKS5 协议

**类别**: 协议

## 概述

SOCKS5 是 SOCKS 协议的第 5 版，用于在客户端和服务器之间建立代理连接。支持 TCP 和 UDP 代理，以及用户名/密码认证。

## 原理

### 连接流程

```
Client                    Server
  |                          |
  |--- 认证请求 ----------->|  (支持的认证方法)
  |<-- 认证响应 ------------|  (选定的认证方法)
  |                          |
  |--- 认证数据 ----------->|  (如果需要)
  |<-- 认证结果 ------------|
  |                          |
  |--- 连接请求 ----------->|  (目标地址)
  |<-- 连接响应 ------------|  (连接状态)
  |                          |
  |<========= 数据 =========>|
```

### 请求格式

```
请求:
  VER: 1 byte (0x05)
  CMD: 1 byte (0x01=CONNECT, 0x02=BIND, 0x03=UDP)
  RSV: 1 byte (0x00)
  ATYP: 1 byte (地址类型)
  DST.ADDR: variable
  DST.PORT: 2 bytes
```

### 地址类型

| ATYP | 说明 |
|------|------|
| 0x01 | IPv4 (4 bytes) |
| 0x03 | 域名 (1 byte length + domain) |
| 0x04 | IPv6 (16 bytes) |

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| wire 命名空间 | 零拷贝解析 | [[protocol/socks5/wire|wire]] |
| stream 类 | relay 状态机 | [[protocol/socks5/stream|stream]] |

## 参考资料

- [RFC 1928 - SOCKS Protocol Version 5](https://tools.ietf.org/html/rfc1928)

## 相关知识

- [[ref/protocol/http-connect|HTTP CONNECT]] — HTTP 代理
