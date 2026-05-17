---
title: MASQUE
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, masque]
---
# MASQUE 协议

MASQUE 是基于 HTTP/3 的 IP-in-IP 隧道协议。

## 协议概述

MASQUE 特性：
- IP-in-IP over HTTP/3
- 使用 Connect-IP 扩展
- 支持远程 DNS 解析
- 支持拥塞控制

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/masque.go` | MASQUE 适配器 |
| `transport/masque/masque.go` | MASQUE 实现 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "masque-proxy"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
```

### IPv6 配置

```yaml
proxies:
  - name: "masque-ipv6"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
    ipv6: fd00::2
```

### 远程 DNS

```yaml
proxies:
  - name: "masque-dns"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
    remote-dns-resolve: true
    dns:
      - 1.1.1.1
```

### 拥塞控制

```yaml
proxies:
  - name: "masque-cc"
    type: masque
    server: server.example.com
    port: 443
    private-key: your-private-key
    public-key: server-public-key
    ip: 10.0.0.2
    congestion-controller: bbr
    cwnd: 32
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `masque` |
| `server` | string | 是 | 服务器地址 |
| `port` | int | 是 | 服务器端口 |
| `private-key` | string | 是 | ECDSA 私钥（Base64） |
| `public-key` | string | 是 | 服务器公钥（Base64） |
| `ip` | string | 是 | 本地 IPv4 |
| `ipv6` | string | 否 | 本地 IPv6 |
| `mtu` | int | 否 | MTU（默认 1280） |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | MASQUE 实现 |
| Pipeline | 完全兼容 | L3 协议 |
| Multiplex | 完全兼容 | HTTP/3 多路复用 |

## 相关文档

- [[wireguard]] - WireGuard 协议
- [[../../dev/tls]] - TLS/QUIC