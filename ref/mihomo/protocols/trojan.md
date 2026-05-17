---
title: Trojan
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, trojan]
---
# Trojan 协议

Trojan 是一种轻量级 TLS 代理协议，通过 TLS 隐藏代理流量。

## 协议概述

Trojan 特性：
- 基于 TLS 的轻量级代理
- 使用密码认证（SHA224）
- 支持 WebSocket 和 gRPC 传输
- 支持 UDP over TCP
- 支持 Shadowsocks 加密层（Trojan-Go）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/trojan.go` | Trojan 适配器 |
| `transport/trojan/trojan.go` | Trojan 协议实现 |
| `transport/gun/gun.go` | gRPC 传输 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "trojan-proxy"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    udp: true
```

### WebSocket 配置

```yaml
proxies:
  - name: "trojan-ws"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    network: ws
    ws-opts:
      path: /trojan
      headers:
        Host: server.example.com
```

### gRPC 配置

```yaml
proxies:
  - name: "trojan-grpc"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    network: grpc
    grpc-opts:
      grpc-service-name: trojan
```

### Reality 配置

```yaml
proxies:
  - name: "trojan-reality"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    reality-opts:
      public-key: xxx
      short-id: xxx
```

### Trojan-Go SS 加密

```yaml
proxies:
  - name: "trojan-ss"
    type: trojan
    server: server.example.com
    port: 443
    password: your-password
    ss-opts:
      enabled: true
      method: aes-128-gcm
      password: ss-password
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `trojan` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `password` | string | 是 | Trojan 密码 |
| `udp` | bool | 否 | 启用 UDP |
| `network` | string | 否 | 传输方式：tcp/ws/grpc |
| `sni` | string | 否 | TLS SNI |
| `skip-cert-verify` | bool | 否 | 跳过证书验证 |
| `fingerprint` | string | 否 | TLS 指纹 |
| `ech-opts` | object | 否 | ECH 配置选项 |
| `reality-opts` | object | 否 | Reality 配置选项（含 public-key、short-id） |
| `client-fingerprint` | string | 否 | TLS 客户端指纹（如 chrome） |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | Trojan 协议标准实现 |
| Pipeline | 完全兼容 | 支持 StreamConnContext |
| Multiplex | 支持 | 通过 gRPC 实现多路复用 |

## 相关文档

- [[../../protocol/trojan]] - Trojan 协议规范
- [[../../protocol/trojan-gfw]] - Trojan-GFW 版本
- [[reality]] - Reality TLS 隐蔽
- [[../transport/gun]] - gRPC 传输