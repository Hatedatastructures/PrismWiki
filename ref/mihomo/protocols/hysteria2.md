---
title: Hysteria2
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, hysteria2]
---
# Hysteria2 协议

Hysteria2 是 Hysteria 协议的升级版本，提供更好的性能和隐蔽性。

## 协议概述

Hysteria2 特性：
- 基于 QUIC 的高性能传输
- Salamander 混淆
- 更简化的配置
- 支持端口跳跃
- 内置拥塞控制

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/hysteria2.go` | Hysteria2 适配器 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "hysteria2-proxy"
    type: hysteria2
    server: server.example.com
    port: 443
    password: your-password
```

### Salamander 混淆

```yaml
proxies:
  - name: "hysteria2-obfs"
    type: hysteria2
    server: server.example.com
    port: 443
    password: your-password
    obfs: salamander
    obfs-password: obfs-string
```

### 端口跳跃

```yaml
proxies:
  - name: "hysteria2-hop"
    type: hysteria2
    server: server.example.com
    ports: 10000-50000
    hop-interval: 30
    password: your-password
```

### 带宽配置

```yaml
proxies:
  - name: "hysteria2-speed"
    type: hysteria2
    server: server.example.com
    port: 443
    password: your-password
    up: "100 Mbps"
    down: "200 Mbps"
    cwnd: 32
```

### 完整配置

```yaml
proxies:
  - name: "hysteria2-full"
    type: hysteria2
    server: server.example.com
    port: 443
    password: your-password
    obfs: salamander
    obfs-password: obfs-string
    sni: server.example.com
    skip-cert-verify: false
    alpn:
      - h3
    fingerprint: chrome
    up: "100 Mbps"
    down: "200 Mbps"
    cwnd: 32
    udp-mtu: 1200
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `hysteria2` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是* | 服务器端口 |
| `ports` | string | 否 | 端口范围 |
| `password` | string | 是 | 认证密码 |
| `obfs` | string | 否 | 混淆类型 |
| `obfs-password` | string | 否 | 混淆密码 |
| `up` | string | 否 | 上传带宽 |
| `down` | string | 否 | 下载带宽 |
| `sni` | string | 否 | TLS SNI |
| `cwnd` | int | 否 | 拥塞窗口 |
| `hop-interval` | int | 否 | 跳跃间隔（秒） |
| `udp-mtu` | int | 否 | UDP MTU（默认 1197） |
| `initial-stream-receive-window` | uint64 | 否 | QUIC 流初始接收窗口 |
| `max-stream-receive-window` | uint64 | 否 | QUIC 流最大接收窗口 |
| `initial-connection-receive-window` | uint64 | 否 | QUIC 连接初始接收窗口 |
| `max-connection-receive-window` | uint64 | 否 | QUIC 连接最大接收窗口 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Hysteria2 协议实现 |
| Pipeline | 支持 | QUIC 流 |
| Multiplex | 完全兼容 | QUIC 内置多路复用 |

## 相关文档

- [[hysteria]] - Hysteria v1 协议
- [[tuic]] - TUIC 协议
- [[../../dev/udp]] - UDP 传输