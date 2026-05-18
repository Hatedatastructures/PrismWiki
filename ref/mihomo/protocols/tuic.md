---
title: TUIC
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, tuic]
---
# TUIC 协议

TUIC 是基于 QUIC 的 UDP 优化代理协议。

## 协议概述

TUIC 特性：
- 基于 QUIC 的 UDP 传输
- 支持多路复用
- 支持 UDP over Stream 模式
- 版本：v4（Token）和 v5（UUID+Password）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/tuic.go` | TUIC 适配器 |
| `transport/tuic/tuic.go` | TUIC 协议实现 |

## YAML 配置示例

### TUIC v4 配置

```yaml
proxies:
  - name: "tuic-v4"
    type: tuic
    server: server.example.com
    port: 443
    token: your-token
    congestion-controller: cubic
    udp-relay-mode: quic
    reduce-rtt: true
```

### TUIC v5 配置

```yaml
proxies:
  - name: "tuic-v5"
    type: tuic
    server: server.example.com
    port: 443
    uuid: your-uuid
    password: your-password
    congestion-controller: bbr
    udp-relay-mode: native
```

### UDP over Stream

```yaml
proxies:
  - name: "tuic-uos"
    type: tuic
    server: server.example.com
    port: 443
    uuid: your-uuid
    password: your-password
    udp-over-stream: true
    udp-over-stream-version: 2
```

### 完整配置

```yaml
proxies:
  - name: "tuic-full"
    type: tuic
    server: server.example.com
    port: 443
    uuid: your-uuid
    password: your-password
    congestion-controller: bbr
    udp-relay-mode: quic
    reduce-rtt: true
    request-timeout: 8000
    heartbeat-interval: 10000
    max-open-streams: 100
    max-udp-relay-packet-size: 1252
    fast-open: true
    skip-cert-verify: false
    alpn:
      - h3
    sni: server.example.com
```

## 拥塞控制

| Controller | 描述 |
|------------|------|
| `cubic` | CUBIC 拥塞控制 |
| `bbr` | BBR 拥塞控制 |
| `new_reno` | New Reno |

## UDP Relay 模式

| Mode | 描述 |
|------|------|
| `quic` | QUIC Datagram |
| `native` | 原生 UDP |

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `tuic` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `token` | string | 否 | v4 Token |
| `uuid` | string | 否 | v5 UUID |
| `password` | string | 否 | v5 Password |
| `congestion-controller` | string | 否 | 拥塞控制 |
| `udp-relay-mode` | string | 否 | UDP 模式 |
| `reduce-rtt` | bool | 否 | 0-RTT |
| `fast-open` | bool | 否 | 快速打开 |
| `ip` | string | 否 | 直接指定服务器 IP（绕过 DNS） |
| `disable-sni` | bool | 否 | 禁用 SNI（需配合 skip-cert-verify） |
| `recv-window-conn` | int | 否 | QUIC 流接收窗口 |
| `recv-window` | int | 否 | QUIC 连接接收窗口 |
| `disable-mtu-discovery` | bool | 否 | 禁用 MTU 发现 |
| `max-datagram-frame-size` | int | 否 | 最大数据帧大小（默认 1252） |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | TUIC 协议实现 |
| Pipeline | 支持 | QUIC 流 |
| Multiplex | 完全兼容 | QUIC 多路复用 |

## 相关文档

- [[hysteria]] - Hysteria 协议
- [[hysteria2]] - Hysteria2 协议
- [[dev/debugging/udp|udp]] - UDP 传输