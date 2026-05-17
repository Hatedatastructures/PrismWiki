---
title: "传输层支持矩阵"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/compatibility
source: "mihomo-Meta/transport/"
tags: [mihomo, transport, matrix, websocket, grpc, quic, mux]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/compatibility/overview]]"
  - "[[ref/mihomo/compatibility/protocol-matrix]]"
  - "[[ref/mihomo/transport/overview]]"
---

# 传输层支持矩阵

**类别**: Mihomo 兼容性 | **模块**: 传输层支持状态

## 概述

本页面详细列出 mihomo 支持的所有传输层类型及其适用协议。

## 传输层总览

| 传输层 | 类型 | 适用协议 | 说明 |
|--------|------|----------|------|
| TCP | 基础 | 所有 | 原生 TCP |
| WebSocket | HTTP | Trojan/VLESS/VMess | WS over TLS |
| gRPC/gUN | HTTP/2 | Trojan/VLESS/VMess | gRPC over TLS |
| HTTP/2 | HTTP/2 | Trojan/VLESS | H2 over TLS |
| QUIC | UDP | Hysteria/TUIC | 基于 UDP |
| Smux | 多路复用 | 所有 TCP | 多路复用 |
| Yamux | 多路复用 | 所有 TCP | 多路复用 |
| h2mux | HTTP/2 Mux | 所有 TCP | HTTP/2 多路复用 |

## WebSocket 传输

```yaml
proxies:
  - name: "ws-node"
    type: vless
    server: server.com
    port: 443
    uuid: "uuid"
    network: ws
    tls: true
    servername: server.com
    ws-opts:
      path: /path
      headers:
        Host: server.com
      max-early-data: 2048
      early-data-header-name: Sec-WebSocket-Protocol
```

| 特性 | 支持 |
|------|------|
| Path | 支持 |
| Headers | 支持 |
| Early Data | 支持 |
| CDN 兼容 | 支持 |

**适用协议**:
- VLESS
- VMess
- Trojan

## gRPC/gUN 传输

```yaml
proxies:
  - name: "grpc-node"
    type: vless
    server: server.com
    port: 443
    uuid: "uuid"
    network: grpc
    tls: true
    servername: server.com
    grpc-opts:
      grpc-service-name: grpc-service
```

| 特性 | 支持 |
|------|------|
| Service Name | 支持 |
| CDN 兼容 | 支持 |
| HTTP/2 | 内置 |

**适用协议**:
- VLESS
- VMess
- Trojan

## HTTP/2 传输

```yaml
proxies:
  - name: "h2-node"
    type: vless
    server: server.com
    port: 443
    uuid: "uuid"
    network: h2
    tls: true
    servername: server.com
    h2-opts:
      host:
        - server.com
      path: /path
```

| 特性 | 支持 |
|------|------|
| Host | 支持 |
| Path | 支持 |
| 多 Host | 支持 |

**适用协议**:
- VLESS
- VMess
- Trojan

## QUIC 传输

QUIC 是内置传输层，无需单独配置:

```yaml
# Hysteria2 使用 QUIC
proxies:
  - name: "hysteria2-node"
    type: hysteria2
    server: server.com
    port: 443
    password: "password"
    # QUIC 内置，无需配置 network

# TUIC 使用 QUIC
proxies:
  - name: "tuic-node"
    type: tuic
    server: server.com
    port: 443
    uuid: "uuid"
    password: "password"
    alpn:
      - h3
    congestion-controller: bbr
```

| 特性 | 支持 |
|------|------|
| ALPN | 支持 |
| 0-RTT | 支持 |
| 拥塞控制 | BBR/Cubic |

**适用协议**:
- Hysteria
- Hysteria2
- TUIC

## Smux 多路复用

```yaml
proxies:
  - name: "smux-node"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    smux:
      enabled: true
      protocol: smux
      max-connections: 4
      min-streams: 4
      max-streams: 0
      padding: false
      statistic: false
      only-tcp: false
```

| 特性 | 支持 |
|------|------|
| 连接池 | 支持 |
| Padding | 支持 |
| 统计 | 支持 |
| TCP Only | 支持 |

**帧开销**: 12 bytes

## Yamux 多路复用

```yaml
proxies:
  - name: "yamux-node"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    smux:
      enabled: true
      protocol: yamux
```

| 特性 | 支持 |
|------|------|
| 连接池 | 支持 |
| 流控制 | 支持 |

**帧开销**: 12 bytes

## h2mux 多路复用

```yaml
proxies:
  - name: "h2mux-node"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    smux:
      enabled: true
      protocol: h2mux
      max-connections: 16
      brutal-opts:
        enabled: true
        up: "100 Mbps"
        down: "200 Mbps"
```

| 特性 | 支持 |
|------|------|
| HTTP/2 | 内置 |
| Brutal | 支持 |
| 高性能 | 推荐 |

**适用场景**: CDN 环境、高性能需求

## TCP Brutal

```yaml
proxies:
  - name: "brutal-node"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    smux:
      enabled: true
      protocol: h2mux
      max-connections: 16
      brutal-opts:
        enabled: true
        up: "100 Mbps"
        down: "200 Mbps"
```

| 特性 | 支持 |
|------|------|
| 激进模式 | 内置 |
| 带宽控制 | 支持 |
| Linux 优化 | 推荐 |

**适用场景**: Linux 高性能环境

## ShadowTLS 传输

```yaml
proxies:
  - name: "shadowtls-node"
    type: ss
    server: server.com
    port: 443
    cipher: 2022-blake3-aes-128-gcm
    password: "password"
    shadow-tls-opts:
      version: 3
      password: "shadowtls-password"
      sni: server.com
```

| 特性 | 支持 |
|------|------|
| TLS 混淆 | 支持 |
| v2/v3 | 支持 |

## 传输层选择建议

| 场景 | 推荐传输层 | 原因 |
|------|------------|------|
| CDN 环境 | WebSocket/gRPC | CDN 兼容 |
| 高性能 | h2mux + Brutal | 性能最优 |
| 多连接 | Smux/Yamux | 连接复用 |
| UDP 需求 | QUIC (Hysteria2) | UDP 优化 |
| 隐蔽需求 | ShadowTLS | TLS 混淆 |

## 传输层与协议对应

| 协议 | 支持的传输层 |
|------|--------------|
| Trojan | TCP, WS, gRPC, H2 |
| VLESS | TCP, WS, gRPC, H2 |
| VMess | TCP, WS, gRPC, H2 |
| Shadowsocks | TCP, 插件 |
| Hysteria2 | QUIC (内置) |
| TUIC | QUIC (内置) |
| WireGuard | UDP |

## 相关链接

- [[overview]] — 兼容性总览
- [[protocol-matrix]] — 协议支持矩阵
- [[ref/mihomo/transport/overview]] — 传输层总览
- [[ref/mihomo/mux/config]] — Mux 配置详解