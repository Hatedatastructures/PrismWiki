---
title: Mihomo 协议参考总览
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, overview]
---

# mihomo 协议参考

mihomo（原 Clash Meta）是一个功能强大的代理核心，支持多种代理协议。本文档汇总 mihomo 支持的所有代理协议及其配置方式。

## 协议分类

### 传输层协议

| 协议 | 类型 | 描述 |
|------|------|------|
| [[socks5]] | TCP/UDP | SOCKS5 协议，支持认证和 TLS |
| [[socks4]] | TCP | SOCKS4/4a 协议，仅 TCP |
| [[http]] | TCP | HTTP CONNECT 代理 |
| [[ssh]] | TCP | SSH 隧道代理 |

### 专用代理协议

| 协议 | 类型 | 描述 |
|------|------|------|
| [[trojan]] | TCP/UDP | Trojan 协议，TLS 加密 |
| [[vless]] | TCP/UDP | VLESS 协议，轻量级 |
| [[vmess]] | TCP/UDP | VMess 协议，V2Ray 核心 |
| [[shadowsocks]] | TCP/UDP | Shadowsocks AEAD |
| [[shadowsocksr]] | TCP/UDP | ShadowsocksR 协议 |
| [[snell]] | TCP/UDP | Snell 协议 |

### 基于 QUIC 的协议

| 协议 | 类型 | 描述 |
|------|------|------|
| [[hysteria]] | TCP/UDP | Hysteria v1，基于 QUIC |
| [[hysteria2]] | TCP/UDP | Hysteria v2，优化版 |
| [[tuic]] | TCP/UDP | TUIC 协议，QUIC 多路复用 |

### VPN 类协议

| 协议 | 类型 | 描述 |
|------|------|------|
| [[wireguard]] | TCP/UDP | WireGuard VPN 协议 |
| [[masque]] | TCP/UDP | IP-in-IP over HTTP/3 |

### 隐蔽性协议

| 协议 | 类型 | 描述 |
|------|------|------|
| [[anytls]] | TCP/UDP | AnyTLS 协议，TLS 隐蔽 |
| [[reality]] | TLS | Reality TLS 隐蔽技术 |
| [[ech]] | TLS | Encrypted Client Hello |
| [[trusttunnel]] | TCP/UDP | TrustTunnel 协议 |

### 新兴协议

| 协议 | 类型 | 描述 |
|------|------|------|
| [[mieru]] | TCP/UDP | Mieru 协议 |
| [[sudoku]] | TCP/UDP | Sudoku 协议 |

### 功能型协议

| 协议 | 类型 | 描述 |
|------|------|------|
| [[direct]] | TCP/UDP | 直接连接 |
| [[reject]] | - | 拒绝连接 |
| [[ref/mihomo/protocols/dns|dns]] | TCP/UDP | DNS 代理 |

## 源码位置

- **适配器**: `adapter/outbound/*.go`
- **传输层**: `transport/*/`

## 相关文档

- [[ref/mihomo/transport/overview|传输层概述]]
- [[ref/mihomo/config|配置参考]]
- [[core/crypto/overview|overview]]