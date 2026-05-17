---
title: "Listeners 概览"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/listener"
tags: [mihomo, listeners, 入站, inbound, 监听器]
created: 2026-05-17
updated: 2026-05-17
related: [mixed, socks, http, tun, redir, tproxy]
---

# Listeners 概览

**类别**: Mihomo 入站监听器参考

## 概述

Listeners（入站监听器）是 Mihomo 接收代理流量的入口点。每种监听器类型支持不同的代理协议和网络模式，客户端通过连接监听器来使用代理服务。

### 监听器类型

| 类型 | 协议 | 说明 |
|------|------|------|
| [[ref/mihomo/listeners/mixed|Mixed]] | HTTP + SOCKS5 | 混合监听，推荐使用 |
| [[ref/mihomo/listeners/socks|SOCKS]] | SOCKS4/4a/5 | SOCKS 代理 |
| [[ref/mihomo/listeners/http|HTTP]] | HTTP CONNECT | HTTP 代理 |
| [[ref/mihomo/listeners/tun|TUN]] | TUN 设备 | 透明代理（系统级） |
| [[ref/mihomo/listeners/redir|Redir]] | iptables REDIR | Linux 透明代理 |
| [[ref/mihomo/listeners/tproxy|TProxy]] | iptables TPROXY | Linux 透明代理 |

## 配置位置

监听器配置在 `listeners` 列表中：

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
    # ...

  - name: tun-in
    type: tun
    # ...
```

## 入站流程

```
入站流程：
┌─────────────────────────────────────────────┐
│                                             │
│  客户端                                      │
│    │                                        │
│    ├── HTTP 请求 ──────→ HTTP Listener      │
│    ├── SOCKS5 请求 ────→ SOCKS Listener    │
│    ├── 系统流量 ────────→ TUN Listener      │
│    ├── iptables REDIR ──→ Redir Listener   │
│    └── iptables TPROXY ─→ TProxy Listener  │
│                                             │
│                      ↓                      │
│                                             │
│                   路由决策                   │
│                      ↓                      │
│                                             │
│                 出站代理节点                │
│                                             │
└─────────────────────────────────────────────┘
```

## 监听器通用配置

### 通用字段

```yaml
listeners:
  - name: listener-name     # 监听器名称
    type: mixed             # 监听器类型
    listen: 0.0.0.0:7890    # 监听地址
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | string | 是 | 监听器名称 |
| type | string | 是 | 监听器类型 |
| listen | string | 是 | 监听地址（host:port） |

### 认证配置

支持用户名密码认证：

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
    users:
      - username: user1
        password: pass1
      - username: user2
        password: pass2
```

### TLS 配置

部分监听器支持 TLS：

```yaml
listeners:
  - name: https-in
    type: http
    listen: 0.0.0.0:443
    tls:
      certificate: /path/to/cert.pem
      private-key: /path/to/key.pem
```

## 平台支持

| 监听器 | Linux | macOS | Windows |
|--------|-------|-------|---------|
| Mixed | 支持 | 支持 | 支持 |
| SOCKS | 支持 | 支持 | 支持 |
| HTTP | 支持 | 支持 | 支持 |
| TUN | 支持 | 支持 | 支持 |
| Redir | 支持 | 不支持 | 不支持 |
| TProxy | 支持 | 不支持 | 不支持 |

## 使用场景

### 本地代理

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 127.0.0.1:7890
```

适用：本地应用代理

### 局域网代理

```yaml
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
    allow-lan: true
```

适用：局域网共享代理

### 透明代理

```yaml
listeners:
  - name: tun-in
    type: tun
    # TUN 配置...
```

适用：全局代理、路由器

## 相关链接

- [[ref/mihomo/listeners/mixed|Mixed 监听器]] — HTTP + SOCKS5 混合监听
- [[ref/mihomo/listeners/socks|SOCKS 监听器]] — SOCKS 代理监听
- [[ref/mihomo/listeners/http|HTTP 监听器]] — HTTP 代理监听
- [[ref/mihomo/listeners/tun|TUN 监听器]] — TUN 设备透明代理
- [[ref/mihomo/listeners/redir|Redir 监听器]] — Linux iptables REDIR
- [[ref/mihomo/listeners/tproxy|TProxy 监听器]] — Linux iptables TPROXY