---
title: Hysteria
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, hysteria]
---
# Hysteria 协议

Hysteria 是基于 QUIC 的高性能代理协议。

## 协议概述

Hysteria v1 特性：
- 基于 QUIC 协议
- Brutal 拥塞控制算法
- 支持端口跳跃
- 支持混淆
- 高性能 UDP 传输

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/hysteria.go` | Hysteria 适配器 |
| `transport/hysteria/core/client.go` | Hysteria 核心客户端 |
| `transport/hysteria/congestion/brutal.go` | Brutal 拥塞控制 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "hysteria-proxy"
    type: hysteria
    server: server.example.com
    port: 443
    auth: your-auth-string
    up: "100 Mbps"
    down: "200 Mbps"
    obfs: obfs-string
```

### 端口跳跃

```yaml
proxies:
  - name: "hysteria-hop"
    type: hysteria
    server: server.example.com
    ports: 10000-50000
    protocol: udp
    auth: your-auth-string
    up: "100 Mbps"
    down: "200 Mbps"
    hop-interval: 10
```

### 完整配置

```yaml
proxies:
  - name: "hysteria-full"
    type: hysteria
    server: server.example.com
    port: 443
    auth-str: your-auth-string
    obfs: obfs-password
    up: "100 Mbps"
    down: "200 Mbps"
    sni: server.example.com
    skip-cert-verify: false
    alpn:
      - hysteria
    recv-window-conn: 15728640
    recv-window: 67108864
    disable-mtu-discovery: false
    fast-open: true
```

## 带宽配置

带宽可使用以下格式：
- `100 Mbps` - Mbps 单位
- `1 Gbps` - Gbps 单位
- `12500000` - Bps 单位（字节/秒）

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `hysteria` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是* | 服务器端口（ports 时可选） |
| `ports` | string | 否 | 端口范围（跳跃） |
| `auth` | string | 否 | Base64 认证 |
| `auth-str` | string | 否 | 字符串认证 |
| `up` | string | 是 | 上传带宽 |
| `down` | string | 是 | 下载带宽 |
| `obfs` | string | 否 | 混淆密码 |
| `sni` | string | 否 | TLS SNI |
| `fast-open` | bool | 否 | 快速打开 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Hysteria 协议实现 |
| Pipeline | 支持 | QUIC 流 |
| Multiplex | 完全兼容 | QUIC 内置多路复用 |

## 相关文档

- [[hysteria2]] - Hysteria v2 协议
- [[tuic]] - TUIC 协议
- [[dev/debugging/udp|udp]] - UDP 传输