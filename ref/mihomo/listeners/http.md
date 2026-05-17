---
title: "HTTP 监听器"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/listener/inbound/http.go"
tags: [mihomo, listeners, http, http-connect, 代理]
created: 2026-05-17
updated: 2026-05-17
related: [overview, mixed, socks]
---

# HTTP 监听器

**类别**: Mihomo 入站监听器

## 概述

HTTP 监听器提供 HTTP CONNECT 代理服务。客户端通过发送 HTTP CONNECT 请求建立代理连接，适用于浏览器、命令行工具等支持 HTTP 代理的应用。

## 基础配置

### 最小配置

```yaml
listeners:
  - name: http-in
    type: http
    listen: 0.0.0.0:7892
```

### 完整配置

```yaml
listeners:
  - name: http-in
    type: http
    listen: 0.0.0.0:7892
    allow-lan: true
    users:
      - username: user1
        password: pass1
    tls:
      certificate: /path/to/cert.pem
      private-key: /path/to/key.pem
```

## 配置参数

### name

```yaml
name: http-in
```

监听器名称。

### type

```yaml
type: http
```

固定值 `http`，表示 HTTP 监听器。

### listen

```yaml
listen: 0.0.0.0:7892
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

HTTP Basic 认证配置。

### tls

```yaml
tls:
  certificate: /path/to/cert.pem
  private-key: /path/to/key.pem
```

TLS 配置，启用 HTTPS 代理。

## HTTP CONNECT 原理

### CONNECT 请求

```http
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
User-Agent: curl/7.64.1
Proxy-Connection: Keep-Alive
```

### CONNECT 响应

```http
HTTP/1.1 200 Connection Established
Proxy-Agent: mihomo
```

### 代理流程

```
HTTP CONNECT 代理流程：
┌─────────────────────────────────────────────┐
│                                             │
│  客户端                        Mihomo        │
│    │                           │            │
│    │── CONNECT example.com:443│            │
│    │                           │            │
│    │←── 200 Connection Est. ──│            │
│    │                           │            │
│    │════ TLS 握手 ═══════════│            │
│    │                           │            │
│    │════ 加密数据流 ═════════│            │
│                                             │
└─────────────────────────────────────────────┘
```

## HTTP 认证

### Basic 认证

```yaml
users:
  - username: admin
    password: secret123
```

客户端连接时需要提供认证：

```http
CONNECT example.com:443 HTTP/1.1
Host: example.com:443
Proxy-Authorization: Basic YWRtaW46c2VjcmV0MTIz
```

### 认证响应

认证失败时返回 407：

```http
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="Proxy"
Content-Length: 0
```

## HTTPS 代理

启用 TLS 后，客户端使用 HTTPS 连接代理：

```yaml
listeners:
  - name: https-in
    type: http
    listen: 0.0.0.0:8443
    tls:
      certificate: /etc/mihomo/cert.pem
      private-key: /etc/mihomo/key.pem
```

客户端配置示例：

```bash
curl --proxy https://proxy.example.com:8443 https://example.com
```

## 配置示例

### 本地 HTTP 代理

```yaml
listeners:
  - name: http-in
    type: http
    listen: 127.0.0.1:7892
```

### 局域网 HTTP 代理

```yaml
listeners:
  - name: http-in
    type: http
    listen: 0.0.0.0:7892
    allow-lan: true
```

### 认证 HTTP 代理

```yaml
listeners:
  - name: http-auth
    type: http
    listen: 0.0.0.0:7892
    allow-lan: true
    users:
      - username: proxyuser
        password: proxypass
```

### HTTPS 代理

```yaml
listeners:
  - name: https-in
    type: http
    listen: 0.0.0.0:8443
    allow-lan: true
    tls:
      certificate: /etc/mihomo/cert.pem
      private-key: /etc/mihomo/key.pem
```

## 与 Mixed 监听器的区别

| 特性 | HTTP 监听器 | Mixed 监听器 |
|------|-------------|--------------|
| HTTP 支持 | 支持 | 支持 |
| SOCKS5 支持 | 不支持 | 支持 |
| 协议检测 | 无 | 自动检测 |
| 适用场景 | 纯 HTTP 环境 | 通用代理 |

建议：大多数情况下使用 Mixed 监听器，仅在需要严格限制为 HTTP 协议时使用 HTTP 监听器。

## 客户端配置

### curl

```bash
export http_proxy=http://127.0.0.1:7892
export https_proxy=http://127.0.0.1:7892
```

### wget

```bash
export HTTP_PROXY=http://127.0.0.1:7892
export HTTPS_PROXY=http://127.0.0.1:7892
```

### Git

```bash
git config --global http.proxy http://127.0.0.1:7892
git config --global https.proxy http://127.0.0.1:7892
```

### 浏览器

系统代理设置：

- HTTP 代理：127.0.0.1
- 端口：7892

## 相关链接

- [[ref/mihomo/listeners/overview|Listeners 概览]] — 监听器总览
- [[ref/mihomo/listeners/mixed|Mixed 监听器]] — 混合监听
- [[ref/mihomo/listeners/socks|SOCKS 监听器]] — SOCKS 代理
- [[ref/protocol/http-proxy-spec|HTTP 代理规范]] — HTTP CONNECT 参考