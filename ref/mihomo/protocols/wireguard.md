---
title: WireGuard
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, wireguard]
---
# WireGuard 协议

WireGuard 是现代 VPN 协议，提供简洁高效的 VPN 连接。

## 协议概述

WireGuard 特性：
- 现代 VPN 协议
- 高性能、低延迟
- 支持多 Peer
- 支持 AmneziaWG 扩展
- 内置加密（ChaCha20-Poly1305）

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/wireguard.go` | WireGuard 适配器 |

## YAML 配置示例

### 基本配置

```yaml
proxies:
  - name: "wg-proxy"
    type: wireguard
    server: server.example.com
    port: 51820
    ip: 10.0.0.2
    ipv6: fd00::2
    private-key: your-private-key
    public-key: server-public-key
    udp: true
```

### AmneziaWG 配置

```yaml
proxies:
  - name: "amnezia-wg"
    type: wireguard
    server: server.example.com
    port: 51820
    ip: 10.0.0.2
    private-key: your-private-key
    public-key: server-public-key
    amnezia-wg-option:
      jc: 3
      jmin: 50
      jmax: 1000
      s1: 75
      s2: 75
      h1: 1234567890
      h2: 1234567890
      h3: 1234567890
      h4: 1234567890
```

### 多 Peer 配置

```yaml
proxies:
  - name: "wg-multi-peer"
    type: wireguard
    private-key: your-private-key
    peers:
      - server: server1.example.com
        port: 51820
        public-key: peer1-public-key
        pre-shared-key: psk1
        allowed-ips:
          - 10.0.0.0/24
      - server: server2.example.com
        port: 51820
        public-key: peer2-public-key
        allowed-ips:
          - 10.1.0.0/24
    ip: 10.0.0.2
```

### 远程 DNS 解析

```yaml
proxies:
  - name: "wg-dns"
    type: wireguard
    server: server.example.com
    port: 51820
    ip: 10.0.0.2
    private-key: your-private-key
    public-key: server-public-key
    remote-dns-resolve: true
    dns:
      - 1.1.1.1
      - 8.8.8.8
```

### Reserved 字段

```yaml
proxies:
  - name: "wg-reserved"
    type: wireguard
    server: server.example.com
    port: 51820
    ip: 10.0.0.2
    private-key: your-private-key
    public-key: server-public-key
    reserved:
      - 100
      - 200
      - 300
```

## 配置字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 代理名称 |
| `type` | string | 是 | 固定为 `wireguard` |
| `server` | string | 是* | 服务器地址 |
| `port` | int | 是* | 服务器端口 |
| `ip` | string | 是 | 本地 IPv4 |
| `ipv6` | string | 否 | 本地 IPv6 |
| `private-key` | string | 是 | 本地私钥 |
| `public-key` | string | 是 | 远端公钥 |
| `pre-shared-key` | string | 否 | PSK |
| `reserved` | []uint8 | 否 | Reserved 字段 |
| `mtu` | int | 否 | MTU（默认 1408） |
| `workers` | int | 否 | 工线程数 |
| `udp` | bool | 否 | 启用 UDP |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | WireGuard VPN 实现 |
| Pipeline | 完全兼容 | L3 协议 |
| Multiplex | 不适用 | VPN 协议 |

## 相关文档

- [[masque]] - MASQUE 协议
- [[ref/crypto/chacha20-poly1305|chacha20-poly1305]] - ChaCha20 加密