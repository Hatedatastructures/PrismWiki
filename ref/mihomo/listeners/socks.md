---
title: "SOCKS 监听器"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/listener/inbound/socks.go"
tags: [mihomo, listeners, socks, socks5, socks4, 代理]
created: 2026-05-17
updated: 2026-05-17
related: [overview, mixed, http]
---

# SOCKS 监听器

**类别**: Mihomo 入站监听器

## 概述

SOCKS 监听器提供 SOCKS4、SOCKS4a 和 SOCKS5 代理服务。相比 Mixed 监听器，SOCKS 监听器提供更精细的 SOCKS 协议控制。

## 基础配置

### 最小配置

```yaml
listeners:
  - name: socks-in
    type: socks
    listen: 0.0.0.0:7891
```

### 完整配置

```yaml
listeners:
  - name: socks-in
    type: socks
    listen: 0.0.0.0:7891
    allow-lan: true
    users:
      - username: user1
        password: pass1
    udp: true
    tls:
      certificate: /path/to/cert.pem
      private-key: /path/to/key.pem
```

## 配置参数

### name

```yaml
name: socks-in
```

监听器名称。

### type

```yaml
type: socks
```

固定值 `socks`，表示 SOCKS 监听器。

### listen

```yaml
listen: 0.0.0.0:7891
```

监听地址，格式为 `host:port`。

### allow-lan

```yaml
allow-lan: true
```

是否允许局域网连接。

### users

```yaml
users:
  - username: user1
    password: pass1
```

SOCKS5 用户认证。仅 SOCKS5 支持，SOCKS4/4a 不支持认证。

### udp

```yaml
udp: true
```

| 值 | 说明 |
|-----|------|
| true | 启用 UDP 支持（SOCKS5） |
| false | 仅 TCP |

### tls

```yaml
tls:
  certificate: /path/to/cert.pem
  private-key: /path/to/key.pem
```

TLS 加密配置。

## SOCKS 协议版本

SOCKS 监听器支持多个版本：

### SOCKS4

```
SOCKS4 请求格式：
┌────────────────────────────────────┐
│ VN: 0x04 (SOCKS4)                  │
│ CD: 0x01 (CONNECT) / 0x02 (BIND)   │
│ DSTPORT: 2 bytes                   │
│ DSTIP: 4 bytes                     │
│ USERID: null-terminated string     │
└────────────────────────────────────┘
```

限制：仅支持 IPv4，无域名支持。

### SOCKS4a

```
SOCKS4a 扩展格式：
┌────────────────────────────────────┐
│ VN: 0x04                           │
│ CD: 0x01 (CONNECT)                 │
│ DSTPORT: 2 bytes                   │
│ DSTIP: 0x00 0x00 0x00 0x0X (非0)  │
│ USERID: null-terminated string     │
│ DOMAIN: null-terminated string    │
└────────────────────────────────────┘
```

扩展：支持域名解析。

### SOCKS5

```
SOCKS5 握手流程：
┌─────────────────────────────────────────────┐
│                                             │
│  Client                          Server      │
│    │                               │         │
│    │─── VER=5, METHODS ──────────→│         │
│    │                               │         │
│    │←── VER=5, METHOD ────────────│         │
│    │                               │         │
│    │─── Auth (if needed) ────────→│         │
│    │                               │         │
│    │←── Auth Result ──────────────│         │
│    │                               │         │
│    │─── CONNECT Request ──────────→│         │
│    │                               │         │
│    │←── CONNECT Response ─────────│         │
│    │                               │         │
│    │═══ DATA (TCP/UDP) ═══════════│         │
│                                             │
└─────────────────────────────────────────────┘
```

支持：TCP、UDP、认证、IPv6、域名。

## 配置示例

### 基础 SOCKS5

```yaml
listeners:
  - name: socks-in
    type: socks
    listen: 127.0.0.1:7891
```

### SOCKS5 + UDP

```yaml
listeners:
  - name: socks-in
    type: socks
    listen: 0.0.0.0:7891
    allow-lan: true
    udp: true
```

### 认证 SOCKS5

```yaml
listeners:
  - name: socks-auth
    type: socks
    listen: 0.0.0.0:7891
    allow-lan: true
    users:
      - username: admin
        password: secret
    udp: true
```

### TLS SOCKS5

```yaml
listeners:
  - name: socks-tls
    type: socks
    listen: 0.0.0.0:7891
    tls:
      certificate: /etc/mihomo/cert.pem
      private-key: /etc/mihomo/key.pem
```

## SOCKS5 UDP 关联

SOCKS5 UDP 支持通过 UDP 关联实现：

```
UDP 关联流程：
┌─────────────────────────────────────────────┐
│                                             │
│  1. 客户端发送 UDP ASSOCIATE 请求          │
│     └── 服务端返回 UDP 转发地址和端口       │
│                                             │
│  2. 客户端通过 UDP 地址发送数据             │
│     └── 数据封装在 SOCKS5 UDP 格式中        │
│                                             │
│  3. 服务端转发 UDP 数据                     │
│     └── 响应同样封装为 SOCKS5 UDP 格式      │
│                                             │
└─────────────────────────────────────────────┘
```

UDP 数据格式：

```
SOCKS5 UDP 封装：
┌────────────────────────────────────┐
│ Reserved: 0x00 0x00                │
│ Fragment: 0x00                     │
│ ATYP: 0x01/0x03/0x04              │
│ DST.ADDR: 变长                     │
│ DST.PORT: 2 bytes                  │
│ DATA: 实际 UDP 数据                │
└────────────────────────────────────┘
```

## 协议对比

| 特性 | SOCKS4 | SOCKS4a | SOCKS5 |
|------|--------|---------|--------|
| TCP | 支持 | 支持 | 支持 |
| UDP | 不支持 | 不支持 | 支持 |
| 认证 | 不支持 | 不支持 | 支持 |
| IPv6 | 不支持 | 不支持 | 支持 |
| 域名 | 不支持 | 支持 | 支持 |

## 相关链接

- [[ref/mihomo/listeners/overview|Listeners 概览]] — 监听器总览
- [[ref/mihomo/listeners/mixed|Mixed 监听器]] — 混合监听
- [[ref/mihomo/listeners/http|HTTP 监听器]] — HTTP 代理
- [[ref/protocol/socks5-spec|SOCKS5 规范]] — RFC 1928 参考