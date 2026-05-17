---
title: "Mixed 监听器"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/listener/inbound/mixed.go"
tags: [mihomo, listeners, mixed, http, socks5, 混合监听]
created: 2026-05-17
updated: 2026-05-17
related: [overview, socks, http]
---

# Mixed 监听器

**类别**: Mihomo 入站监听器

## 概述

Mixed 监听器同时支持 HTTP 和 SOCKS5 协议，是 Mihomo 推荐的默认监听器类型。客户端可以使用任意协议连接。

## 基础配置

### 最小配置

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
```

### 完整配置

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
    allow-lan: true
    bind-address: "*"
    users:
      - username: user1
        password: pass1
    tls:
      certificate: /path/to/cert.pem
      private-key: /path/to/key.pem
    sniffer:
      enable: true
```

## 配置参数

### name

```yaml
name: mixed-in
```

监听器名称，用于标识和日志。

### type

```yaml
type: mixed
```

固定值 `mixed`，表示混合监听器。

### listen

```yaml
listen: 0.0.0.0:7890
```

| 格式 | 示例 | 说明 |
|------|------|------|
| host:port | `0.0.0.0:7890` | 监听所有接口 |
| host:port | `127.0.0.1:7890` | 仅本地 |
| host:port | `192.168.1.1:7890` | 指定接口 |

### allow-lan

```yaml
allow-lan: true
```

| 值 | 说明 |
|-----|------|
| true | 允许局域网连接 |
| false | 仅本地连接 |

### bind-address

```yaml
bind-address: "*"
```

绑定地址，`*` 表示所有接口。

### users

```yaml
users:
  - username: user1
    password: pass1
  - username: user2
    password: pass2
```

用户认证配置。设置后需要认证才能使用代理。

### tls

```yaml
tls:
  certificate: /path/to/cert.pem
  private-key: /path/to/key.pem
```

TLS 配置，启用 HTTPS/SOCKS5-TLS。

| 字段 | 说明 |
|------|------|
| certificate | 证书文件路径 |
| private-key | 私钥文件路径 |

### sniffer

```yaml
sniffer:
  enable: true
```

启用入站流量嗅探。

## 协议检测

Mixed 监听器自动检测连接协议：

```
协议检测流程：
┌─────────────────────────────────────────────┐
│                                             │
│  客户端连接                                  │
│      │                                      │
│      ▼                                      │
│  读取首字节                                  │
│      │                                      │
│      ├── 0x05 (SOCKS5) ──→ SOCKS5 处理     │
│      │                                       │
│      ├── HTTP 方法 ──────→ HTTP 处理        │
│      │   (GET/POST/CONNECT/...)             │
│      │                                       │
│      └── 其他 ───────────→ 拒绝连接         │
│                                             │
└─────────────────────────────────────────────┘
```

## 配置示例

### 本地代理

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 127.0.0.1:7890
```

### 局域网代理

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
    allow-lan: true
```

### 认证代理

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
    allow-lan: true
    users:
      - username: admin
        password: secret123
```

### TLS 加密代理

```yaml
listeners:
  - name: mixed-tls
    type: mixed
    listen: 0.0.0.0:8443
    allow-lan: true
    tls:
      certificate: /etc/mihomo/cert.pem
      private-key: /etc/mihomo/key.pem
```

### 完整配置

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
    allow-lan: true
    bind-address: "*"
    users:
      - username: user1
        password: pass1
    tls:
      certificate: /path/to/cert.pem
      private-key: /path/to/key.pem
    sniffer:
      enable: true
```

## 兼容性

| 客户端类型 | 协议 | 支持 |
|-----------|------|------|
| 浏览器 | HTTP | 支持 |
| curl | HTTP | 支持 |
| Shadowsocks 客户端 | SOCKS5 | 支持 |
| V2Ray 客户端 | SOCKS5/HTTP | 支持 |
| 任意 SOCKS5 客户端 | SOCKS5 | 支持 |

## 相关链接

- [[ref/mihomo/listeners/overview|Listeners 概览]] — 监听器总览
- [[ref/mihomo/listeners/socks|SOCKS 监听器]] — SOCKS 代理
- [[ref/mihomo/listeners/http|HTTP 监听器]] — HTTP 代理
- [[ref/mihomo/sniffing/overview|Sniffing 概览]] — 流量嗅探