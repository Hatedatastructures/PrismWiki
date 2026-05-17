---
title: AnyTLS
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, anytls]
---
# AnyTLS 协议

AnyTLS 是一种隐蔽性 TLS 代理协议。

## 协议概述

AnyTLS 特性：
- TLS 隐蔽性代理
- 支持多空闲会话
- 支持 UDP over TCP
- 支持客户端指纹

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/anytls.go` | AnyTLS 适配器 |
| `transport/anytls/client.go` | AnyTLS 客户端 |
| `transport/anytls/session/` | AnyTLS 会话管理 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "anytls-proxy"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
```

### 完整配置

```yaml
proxies:
  - name: "anytls-full"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
    sni: server.example.com
    skip-cert-verify: false
    fingerprint: chrome
    client-fingerprint: chrome
    alpn:
      - h2
      - http/1.1
    idle-session-check-interval: 30
    idle-session-timeout: 60
    min-idle-session: 1
    udp: true
```

### ECH 配置

```yaml
proxies:
  - name: "anytls-ech"
    type: anytls
    server: server.example.com
    port: 443
    password: your-password
    ech-opts:
      enable: true
      config: base64-encoded-ech-config
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `anytls` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `password` | string | 是 | 认证密码 |
| `sni` | string | 否 | TLS SNI |
| `fingerprint` | string | 否 | TLS 指纹 |
| `client-fingerprint` | string | 否 | 客户端指纹 |
| `idle-session-check-interval` | int | 否 | 空闲检查间隔 |
| `idle-session-timeout` | int | 否 | 空闲超时 |
| `min-idle-session` | int | 否 | 最小空闲会话 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | AnyTLS 协议实现 |
| Pipeline | 支持 | TLS 流 |
| Stealth | 完全兼容 | TLS 隐蔽技术 |

## 相关文档

- [[reality]] - Reality TLS 隐蔽
- [[ech]] - ECH 加密 Client Hello
- [[../../stealth/anytls]] - AnyTLS 隐蔽技术