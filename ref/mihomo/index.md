---
title: "Mihomo 配置参考"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta"
tags: [mihomo, 配置, 参考, clash-meta]
created: 2026-05-17
updated: 2026-05-17
related: [protocols, transport, mux, sniffing, listeners, provider, ntp, experimental, script]
---

# Mihomo 配置参考

**类别**: 参考资料

## 概述

Mihomo（原 Clash Meta）是一个功能强大的代理核心，支持多种代理协议和高级功能。本文档汇总 Mihomo 配置参考的所有文档。

## 文档目录

### [[ref/mihomo/protocols|协议支持]]

Mihomo 支持的代理协议：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/protocols/overview|overview]] | 协议概览 |
| [[ref/mihomo/protocols/socks5|socks5]] | SOCKS5 协议 |
| [[ref/mihomo/protocols/socks4|socks4]] | SOCKS4/4a 协议 |
| [[ref/mihomo/protocols/http|http]] | HTTP CONNECT |
| [[ref/mihomo/protocols/trojan|trojan]] | Trojan 协议 |
| [[ref/mihomo/protocols/vless|vless]] | VLESS 协议 |
| [[ref/mihomo/protocols/vmess|vmess]] | VMess 协议 |
| [[ref/mihomo/protocols/shadowsocks|shadowsocks]] | Shadowsocks |
| [[ref/mihomo/protocols/shadowsocksr|shadowsocksr]] | SSR 协议 |
| [[ref/mihomo/protocols/snell|snell]] | Snell 协议 |
| [[ref/mihomo/protocols/ssh|ssh]] | SSH 隧道 |
| [[ref/mihomo/protocols/hysteria|hysteria]] | Hysteria v1 |
| [[ref/mihomo/protocols/hysteria2|hysteria2]] | Hysteria v2 |
| [[ref/mihomo/protocols/tuic|tuic]] | TUIC 协议 |
| [[ref/mihomo/protocols/wireguard|wireguard]] | WireGuard VPN |
| [[ref/mihomo/protocols/anytls|anytls]] | AnyTLS 协议 |
| [[ref/mihomo/protocols/reality|reality]] | Reality TLS |
| [[ref/mihomo/protocols/ech|ech]] | ECH 扩展 |
| [[ref/mihomo/protocols/trusttunnel|trusttunnel]] | TrustTunnel |
| [[ref/mihomo/protocols/mieru|mieru]] | Mieru 协议 |
| [[ref/mihomo/protocols/masque|masque]] | MASQUE 协议 |
| [[ref/mihomo/protocols/sudoku|sudoku]] | Sudoku 协议 |
| [[ref/mihomo/protocols/direct|direct]] | 直连协议 |
| [[ref/mihomo/protocols/reject|reject]] | 拒绝协议 |
| [[ref/mihomo/protocols/dns|dns]] | DNS 代理 |

### [[ref/mihomo/transport|传输层]]

传输层插件配置：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/transport/overview|overview]] | 传输层概览 |
| [[ref/mihomo/transport/shadowtls|shadowtls]] | ShadowTLS |
| [[ref/mihomo/transport/restls|restls]] | RestLS |
| [[ref/mihomo/transport/v2ray-plugin|v2ray-plugin]] | V2Ray Plugin |
| [[ref/mihomo/transport/simple-obfs|simple-obfs]] | Simple Obfs |
| [[ref/mihomo/transport/gost-plugin|gost-plugin]] | GOST Plugin |
| [[ref/mihomo/transport/kcptun|kcptun]] | KCP Tun |
| [[ref/mihomo/transport/gun|gun]] | gRPC Transport |

### [[ref/mihomo/mux|多路复用]]

多路复用协议配置：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/mux/overview|overview]] | Mux 概览 |
| [[ref/mihomo/mux/smux|smux]] | Smux 协议 |
| [[ref/mihomo/mux/yamux|yamux]] | Yamux 协议 |
| [[ref/mihomo/mux/singmux|singmux]] | Sing-Mux |
| [[ref/mihomo/mux/config|config]] | Mux 配置参数 |

### [[ref/mihomo/sniffing|流量嗅探]]

嗅探功能配置：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/sniffing/overview|overview]] | Sniffing 概览 |
| [[ref/mihomo/sniffing/enable|enable]] | 启用嗅探 |
| [[ref/mihomo/sniffing/sniffing-types|sniffing-types]] | 嗅探类型 |
| [[ref/mihomo/sniffing/ports|ports]] | 端口配置 |

### [[ref/mihomo/listeners|入站监听器]]

监听器类型配置：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/listeners/overview|overview]] | Listeners 概览 |
| [[ref/mihomo/listeners/mixed|mixed]] | Mixed 监听器 |
| [[ref/mihomo/listeners/socks|socks]] | SOCKS 监听器 |
| [[ref/mihomo/listeners/http|http]] | HTTP 监听器 |
| [[ref/mihomo/listeners/tun|tun]] | TUN 监听器 |
| [[ref/mihomo/listeners/redir|redir]] | Redir 监听器 |
| [[ref/mihomo/listeners/tproxy|tproxy]] | TProxy 监听器 |

### [[ref/mihomo/provider|资源提供者]]

外部资源管理：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/provider/overview|overview]] | Provider 概览 |
| [[ref/mihomo/provider/proxy-provider|proxy-provider]] | Proxy Provider |
| [[ref/mihomo/provider/rule-provider|rule-provider]] | Rule Provider |
| [[ref/mihomo/provider/health-check|health-check]] | Health Check |
| [[ref/mihomo/provider/override|override]] | Override 覆盖 |

### [[ref/mihomo/ntp|时间同步]]

NTP 配置：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/ntp/overview|overview]] | NTP 概览 |
| [[ref/mihomo/ntp/enable|enable]] | 启用 NTP |
| [[ref/mihomo/ntp/server|server]] | NTP 服务器 |

### [[ref/mihomo/experimental|实验性功能]]

实验性配置：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/experimental/overview|overview]] | Experimental 概览 |
| [[ref/mihomo/experimental/quic-go|quic-go]] | quic-go 配置 |
| [[ref/mihomo/experimental/udp-over-tcp|udp-over-tcp]] | UDP over TCP |

### [[ref/mihomo/script|脚本功能]]

脚本扩展：

| 文档 | 内容 |
|------|------|
| [[ref/mihomo/script/overview|overview]] | Script 概览 |
| [[ref/mihomo/script/shortcuts|shortcuts]] | Shortcuts |
| [[ref/mihomo/script/rule-script|rule-script]] | Rule Script |

## 配置结构

```yaml
# Mihomo 配置结构
# 参考各模块文档了解详细配置

# 基础配置
port: 7890
socks-port: 7891
mixed-port: 7890
allow-lan: true

# DNS 配置
dns:
  enable: true
  # 参见 DNS 模块文档

# 嗅探配置
sniffer:
  enable: true
  # 参见 ref/mihomo/sniffing

# 入站监听器
listeners:
  - name: mixed-in
    type: mixed
    listen: 0.0.0.0:7890
    # 参见 ref/mihomo/listeners

# NTP 配置
ntp:
  enable: true
  server: time.apple.com
  # 参见 ref/mihomo/ntp

# 实验性功能
experimental:
  quic-go: true
  udp-over-tcp: false
  # 参见 ref/mihomo/experimental

# 脚本配置
script:
  shortcuts:
    quic: "udp-over-tcp=true"
  # 参见 ref/mihomo/script

# 代理提供者
proxy-providers:
  my-provider:
    type: http
    # 参见 ref/mihomo/provider

# 规则提供者
rule-providers:
  reject:
    type: http
    # 参见 ref/mihomo/provider

# 代理节点
proxies:
  - name: ss-node
    type: ss
    # 参见 ref/mihomo/protocols

# 代理组
proxy-groups:
  - name: auto
    type: url-test

# 规则
rules:
  - RULE-SET,reject,REJECT
```

## 相关链接

- [[ref/overview|参考资料概览]] — 参考资料总览
- [[ref/protocol/overview|协议规范]] — 协议规范参考
- [[ref/crypto/overview|密码学原理]] — 加密算法参考
- [[ref/network/overview|网络基础]] — 网络技术参考