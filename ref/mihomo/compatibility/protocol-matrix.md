---
title: "协议支持矩阵"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/compatibility
source: "mihomo-Meta/adapter/outbound/"
tags: [mihomo, protocol, matrix, support, socks5, trojan, vless]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/compatibility/overview]]"
  - "[[ref/mihomo/compatibility/features]]"
  - "[[ref/mihomo/protocols/overview]]"
---

# 协议支持矩阵

**类别**: Mihomo 兼容性 | **模块**: 代理协议支持状态

## 概述

本页面详细列出 mihomo 支持的所有代理协议及其功能特性。

## 协议总览

| 协议 | TCP | UDP | TLS | 多路复用 | 说明 |
|------|-----|-----|-----|----------|------|
| SOCKS5 | 支持 | 支持 | 支持 | 支持 | 标准代理协议 |
| SOCKS4 | 支持 | 不支持 | 不支持 | 不支持 | 旧版协议 |
| HTTP | 支持 | 不支持 | 支持 | 不支持 | HTTP CONNECT |
| Trojan | 支持 | 支持 | 支持 | 支持 | TLS 加密 |
| VLESS | 支持 | 支持 | 支持 | 支持 | 轻量协议 |
| VMess | 支持 | 支持 | 支持 | 支持 | V2Ray 核心 |
| Shadowsocks | 支持 | 支持 | 支持 | 支持 | AEAD 版本 |
| ShadowsocksR | 支持 | 支持 | 不支持 | 不支持 | SSR 协议 |
| Snell | 支持 | 支持 | 支持 | 支持 | Snell v3 |
| SSH | 支持 | 支持 | 不支持 | 不支持 | SSH 隧道 |

## 基于 QUIC 的协议

| 协议 | TCP | UDP | 特点 | 说明 |
|------|-----|-----|------|------|
| Hysteria | 支持 | 支持 | QUIC | Hysteria v1 |
| Hysteria2 | 支持 | 支持 | QUIC | 推荐版本 |
| TUIC | 支持 | 支持 | QUIC | 多路复用 |

## VPN 类协议

| 协议 | TCP | UDP | 特点 | 说明 |
|------|-----|-----|------|------|
| WireGuard | 支持 | 支持 | VPN | 现代 VPN |
| MASQUE | 支持 | 支持 | HTTP/3 | IP-in-IP |

## 隐蔽性协议

| 协议 | TCP | UDP | 特点 | 说明 |
|------|-----|-----|------|------|
| AnyTLS | 支持 | 支持 | TLS | TLS 特征隐蔽 |
| Reality | 支持 | 支持 | TLS | TLS 隐蔽技术 |
| ECH | 支持 | 支持 | TLS | 加密 Client Hello |
| TrustTunnel | 支持 | 支持 | TLS | 新协议 |
| Mieru | 支持 | 支持 | 多种 | 新协议 |
| Sudoku | 支持 | 支持 | 多种 | 新协议 |

## 功能型协议

| 协议 | 功能 | 说明 |
|------|------|------|
| Direct | 直连 | 无代理 |
| Reject | 拒绝 | 拒绝连接 |
| DNS | DNS | DNS 代理 |
| Pass | 透传 | 透传流量 |

## 协议详细功能

### SOCKS5

```yaml
proxies:
  - name: "socks5-node"
    type: socks5
    server: server.com
    port: 1080
    # 可选认证
    username: "user"
    password: "pass"
    # TLS 支持
    tls: true
    sni: server.com
    skip-cert-verify: false
    # UDP 支持
    udp: true
```

| 特性 | 支持 |
|------|------|
| 认证 | 用户名/密码 |
| TLS | 支持 |
| UDP | 支持 |
| Mux | 支持 |

### Trojan

```yaml
proxies:
  - name: "trojan-node"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    sni: server.com
    skip-cert-verify: false
    udp: true
    # 多路复用
    smux:
      enabled: true
      protocol: h2mux
```

| 特性 | 支持 |
|------|------|
| TLS | 必需 |
| UDP | 支持 |
| Mux | 支持 |
| Brutal | 支持 |

### VLESS

```yaml
proxies:
  - name: "vless-node"
    type: vless
    server: server.com
    port: 443
    uuid: "uuid"
    flow: xtls-rprx-vision
    tls: true
    servername: server.com
    # Reality 支持
    reality-opts:
      public-key: "xxx"
      short-id: "xxx"
    udp: true
```

| 特性 | 支持 |
|------|------|
| TLS | 支持 |
| Reality | 支持 |
| XTLS | 支持 |
| UDP | 支持 |
| Mux | 支持 |

### Hysteria2

```yaml
proxies:
  - name: "hysteria2-node"
    type: hysteria2
    server: server.com
    port: 443
    password: "password"
    sni: server.com
    skip-cert-verify: false
    up: "100 Mbps"
    down: "200 Mbps"
```

| 特性 | 支持 |
|------|------|
| QUIC | 内置 |
| UDP | 支持 |
| Brutal | 内置 |
| 拥塞控制 | BBR |

### TUIC

```yaml
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
    sni: server.com
    skip-cert-verify: false
    udp-relay-mode: native
    reduce-rtt: true
```

| 特性 | 支持 |
|------|------|
| QUIC | 内置 |
| UDP | 支持 |
| 多路复用 | 内置 |
| 拥塞控制 | BBR/Cubic |

### WireGuard

```yaml
proxies:
  - name: "wireguard-node"
    type: wireguard
    server: server.com
    port: 51820
    ip: 10.0.0.2
    ipv6: fd01:5ca1:ab1e::2
    private-key: "privateKey"
    public-key: "publicKey"
    udp: true
    # 可选预共享密钥
    pre-shared-key: "psk"
```

| 特性 | 支持 |
|------|------|
| VPN | 内置 |
| UDP | 必需 |
| 预共享密钥 | 支持 |

### Shadowsocks

```yaml
proxies:
  - name: "ss-node"
    type: ss
    server: server.com
    port: 8388
    cipher: 2022-blake3-aes-128-gcm
    password: "password"
    udp: true
    # 插件
    plugin: obfs
    plugin-opts:
      mode: tls
      host: server.com
```

| 特性 | 支持 |
|------|------|
| AEAD | 必需 |
| UDP | 支持 |
| 插件 | 支持 |
| Mux | 支持 |

### Shadowsocks 2022

```yaml
proxies:
  - name: "ss2022-node"
    type: ss
    server: server.com
    port: 443
    cipher: 2022-blake3-aes-128-gcm
    password: "base64Key"
    udp: true
```

| 特性 | 支持 |
|------|------|
| Blake3 | 内置 |
| UDP | 支持 |
| PSK | 支持 |

## 协议选择建议

| 场景 | 推荐协议 | 原因 |
|------|----------|------|
| 日常使用 | Trojan | 简单可靠 |
| 隐蔽需求 | VLESS + Reality | TLS 特征隐蔽 |
| 高性能 | Hysteria2 | QUIC 性能好 |
| 游戏/UDP | Hysteria2/TUIC | UDP 优化 |
| VPN 需求 | WireGuard | 标准 VPN |

## 相关链接

- [[overview]] — 兼容性总览
- [[features]] — 功能支持矩阵
- [[ref/mihomo/protocols/overview]] — 协议总览
- [[ref/mihomo/config/full-example.yaml]] — 完整配置示例