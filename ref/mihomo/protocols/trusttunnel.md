---
title: TrustTunnel
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, trusttunnel]
---
# TrustTunnel 协议

TrustTunnel 是一种基于 TLS/QUIC 的隐蔽代理协议。

## 协议概述

TrustTunnel 特性：
- 支持 TLS 和 QUIC 传输
- 支持用户认证
- 支持 UDP
- 支持健康检查
- 支持 ECH

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/trusttunnel.go` | TrustTunnel 适配器 |
| `transport/trusttunnel/trusttunnel.go` | TrustTunnel 实现 |

## YAML 配置示例

### TLS 模式

```yaml
proxies:
  - name: "trusttunnel-tls"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    udp: true
```

### QUIC 模式

```yaml
proxies:
  - name: "trusttunnel-quic"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    quic: true
    congestion-controller: bbr
    cwnd: 32
```

### 健康检查

```yaml
proxies:
  - name: "trusttunnel-health"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    health-check: true
```

### 完整配置

```yaml
proxies:
  - name: "trusttunnel-full"
    type: trusttunnel
    server: server.example.com
    port: 443
    username: user
    password: pass
    sni: server.example.com
    skip-cert-verify: false
    fingerprint: chrome
    client-fingerprint: chrome
    alpn:
      - h2
    udp: true
    quic: false
    health-check: true
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `trusttunnel` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `username` | string | 否 | 用户名 |
| `password` | string | 否 | 密码 |
| `udp` | bool | 否 | 启用 UDP |
| `quic` | bool | 否 | 使用 QUIC |
| `health-check` | bool | 否 | 健康检查 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | TrustTunnel 实现 |
| Pipeline | 完全兼容 | TLS/QUIC 流 |
| Stealth | 完全兼容 | TLS 隐蔽 |

## 相关文档

- [[anytls]] - AnyTLS 协议
- [[ech]] - ECH 加密
- [[core/stealth/trusttunnel|trusttunnel]] - TrustTunnel 详细文档