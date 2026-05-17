---
title: "quic-go 配置"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/experimental"
tags: [mihomo, experimental, quic-go, quic, hysteria, tuic]
created: 2026-05-17
updated: 2026-05-17
related: [overview, udp-over-tcp]
---

# quic-go 配置

**类别**: Mihomo 实验性功能

## 概述

quic-go 配置控制 Mihomo 对 QUIC 协议的支持方式。启用后，Mihomo 使用 quic-go 库实现 QUIC 相关功能，支持 Hysteria、TUIC、MASQUE 等协议。

## 基础配置

```yaml
experimental:
  quic-go: true
```

## 配置参数

### quic-go

```yaml
quic-go: true
```

| 值 | 说明 |
|-----|------|
| true | 启用 quic-go 实现 |
| false | 使用其他实现（默认） |

## QUIC 协议支持

启用 quic-go 后支持的协议：

| 协议 | 说明 | 类型 |
|------|------|------|
| Hysteria | Hysteria v1 | 基于 QUIC |
| Hysteria2 | Hysteria v2 | 基于 QUIC |
| TUIC | TUIC v5 | 基于 QUIC |
| MASQUE | IP-in-IP over HTTP/3 | 基于 QUIC |
| WireGuard | WireGuard over QUIC | VPN 类 |

## quic-go 库

quic-go 是 Go 语言实现的 QUIC 协议库。

### 特性

| 特性 | 说明 |
|------|------|
| QUIC v1 | RFC 9000 支持 |
| HTTP/3 | RFC 9114 支持 |
| 0-RTT | 早期数据支持 |
| 多路复用 | 流级别控制 |
| 连接迁移 | 地址变更支持 |

### 版本支持

| QUIC 版本 | 支持 |
|-----------|------|
| QUIC v1 (RFC 9000) | 支持 |
| QUIC v2 | 支持 |
| draft-29 | 支持 |
| draft-27 | 支持 |

## 配置示例

### 启用 quic-go

```yaml
experimental:
  quic-go: true
```

### Hysteria2 配置

```yaml
experimental:
  quic-go: true

proxies:
  - name: hysteria2-node
    type: hysteria2
    server: hysteria.server.com
    port: 443
    password: "your-password"
    sni: hysteria.server.com
    skip-cert-verify: false
```

### TUIC 配置

```yaml
experimental:
  quic-go: true

proxies:
  - name: tuic-node
    type: tuic
    server: tuic.server.com
    port: 443
    uuid: "your-uuid"
    password: "your-password"
    congestion-controller: bbr
    sni: tuic.server.com
```

### WireGuard over QUIC

```yaml
experimental:
  quic-go: true

proxies:
  - name: wg-quic
    type: wireguard
    server: wg.server.com
    port: 443
    ip: 10.0.0.2
    ipv6: fd01:5ca1:ab1e::2
    private-key: "private-key"
    public-key: "public-key"
    reserved: [0, 0, 0]
    # 通过 QUIC 传输
```

## QUIC 连接流程

```
QUIC 连接流程：
┌─────────────────────────────────────────────┐
│                                             │
│  客户端 (Mihomo)                            │
│      │                                      │
│      │ Initial Packet (包含 ClientHello)   │
│      ▼                                      │
│  QUIC 服务器                                 │
│      │                                      │
│      │ Initial Packet (Handshake)          │
│      ▼                                      │
│  TLS 1.3 握手                               │
│      │                                      │
│      │ 1-RTT 完成                           │
│      ▼                                      │
│  数据传输                                    │
│      │                                      │
│      │ Stream 1: TCP 代理                   │
│      │ Stream 2: UDP 代理                   │
│      │ ...                                  │
│                                             │
└─────────────────────────────────────────────┘
```

## QUIC vs TCP

| 特性 | QUIC | TCP |
|------|------|-----|
| 连接建立 | 0-RTT/1-RTT | 多次握手 |
| 多路复用 | 内置流 | 需外部 Mux |
| 连接迁移 | 支持 | 不支持 |
| 拥塞控制 | 可配置 | 固定 |
| NAT 穿透 | 较好 | 一般 |

## 使用建议

### 启用场景

推荐启用 quic-go：

- 使用 Hysteria/Hysteria2 协议
- 使用 TUIC 协议
- 需要 QUIC 多路复用
- 追求低延迟连接

### 禁用场景

可以禁用：

- 不使用 QUIC 协议
- 配置稳定性优先
- 其他实现兼容性

## 排错指南

### QUIC 连接失败

检查项：

1. quic-go 是否启用
2. 协议配置是否正确
3. 服务端是否支持 QUIC
4. UDP 端口是否开放

### 性能问题

调优建议：

- 配置合适的 congestion-controller
- 调整 UDP buffer 大小
- 检查网络延迟

### 兼容性

注意：

- quic-go 版本可能与服务端版本不同
- draft 版本号需要匹配
- 检查协议版本兼容性

## 相关链接

- [[ref/mihomo/experimental/overview|Experimental 概览]] — 实验性功能总览
- [[ref/mihomo/experimental/udp-over-tcp|udp-over-tcp]] — UDP over TCP 配置
- [[ref/mihomo/protocols/hysteria|Hysteria]] — Hysteria 协议详解
- [[ref/mihomo/protocols/tuic|TUIC]] — TUIC 协议详解
- [[ref/protocol/quic-basics|QUIC 基础]] — QUIC 协议参考