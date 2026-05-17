---
title: "mihomo 实现参考"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/implementation
source: "mihomo-Meta/"
tags: [mihomo, implementation, udp-relay, tcp-concurrent, keepalive, authentication]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/implementation/udp-relay]]"
  - "[[ref/mihomo/implementation/tcp-concurrent]]"
  - "[[ref/mihomo/implementation/keepalive]]"
  - "[[ref/mihomo/implementation/authentication]]"
---

# mihomo 实现参考

**类别**: Mihomo 实现机制 | **模块**: 核心功能实现详解

## 概述

本章节深入解析 mihomo 核心功能的实现原理，包括 UDP 中继、TCP 并发、连接保活、认证机制等。

### 核心实现模块

| 模块 | 描述 |
|------|------|
| UDP Relay | UDP 数据中继实现 |
| TCP Concurrent | TCP 并发优化机制 |
| Keepalive | 连接保活实现 |
| Authentication | 认证机制实现 |

## 子页面导航

| 页面 | 描述 |
|------|------|
| [[udp-relay]] | UDP 中继实现详解 |
| [[tcp-concurrent]] | TCP 并发优化详解 |
| [[keepalive]] | 连接保活机制详解 |
| [[authentication]] | 认证机制实现详解 |

## 实现架构

### 核心组件

```
┌─────────────────────────────────────────────────────────────────────┐
│                        mihomo 核心架构                                │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐
│     入站层          │     │     出站层          │
│  (Inbound)          │────▶│  (Outbound)         │
│                     │     │                     │
│  HTTP/SOCKS5/TUN    │     │  协议适配器          │
└─────────────────────┘     │  Trojan/VLESS/SS   │
                            └─────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐
│     规则引擎        │     │     DNS 模块        │
│  (Rule Engine)      │     │  (DNS Module)      │
│                     │     │                     │
│  域名/IP/GeoIP     │     │  fake-ip/cache     │
│  GeoSite/规则集    │     │  DoH/DoT           │
└─────────────────────┘     └─────────────────────┘

┌─────────────────────┐     ┌─────────────────────┐
│     传输层          │     │     连接管理        │
│  (Transport)        │     │  (Connection)      │
│                     │     │                     │
│  WS/gRPC/H2        │     │  Pool/Keepalive    │
│  Smux/Yamux        │     │  Concurrent        │
└─────────────────────┘     └─────────────────────┘
```

### 关键实现位置

| 功能 | 源码位置 |
|------|----------|
| 协议适配器 | `adapter/outbound/*.go` |
| 入站处理 | `adapter/inbound/*.go` |
| 规则引擎 | `component/rule/*.go` |
| DNS 模块 | `component/dns/*.go` |
| 传输层 | `transport/*/` |
| TUN 实现 | `transport/tun/` |

## 实现特性对比

| 特性 | mihomo | Prism |
|------|--------|-------|
| UDP Relay | Fullcone/Native | Native |
| TCP Concurrent | 支持 | 支持 |
| Keepalive | 内置 | 独立模块 |
| 认证 | 协议内置 | 模块化 |
| fake-ip | 原生 | 不支持 |

## 性能优化

### mihomo 性能优化机制

1. **TCP Concurrent**: 多服务器并发尝试
2. **Unified Delay**: 统一延迟计算
3. **TCP Brutal**: 激进多路复用
4. **Lazy 测速**: 按需测速
5. **连接池**: 减少握手开销

### 配置示例

```yaml
# 性能优化配置
tcp-concurrent: true        # TCP 并发
unified-delay: true         # 统一延迟
keep-alive-interval: 30     # 保活间隔

# Brutal 配置
smux:
  brutal-opts:
    enabled: true
    up: "100 Mbps"
    down: "200 Mbps"
```

## 源码结构

```
mihomo/
├── adapter/
│   ├── outbound/           # 出站协议适配器
│   │   ├── trojan.go
│   │   ├── vless.go
│   │   ├── vmess.go
│   │   ├── ss.go
│   │   ├── hysteria2.go
│   │   └── ...
│   └── inbound/            # 入站处理
│
├── transport/              # 传输层实现
│   ├── smux/
│   ├── yamux/
│   ├── tun/
│   └── ...
│
├── component/
│   ├── dns/                # DNS 模块
│   ├── rule/               # 规则引擎
│   └── sniffer/            # 嗅探模块
│
└── config/                 # 配置解析
```

## 相关链接

- [[udp-relay]] — UDP 中继详解
- [[tcp-concurrent]] — TCP 并发详解
- [[keepalive]] — 保活机制详解
- [[authentication]] — 认证机制详解
- [[ref/mihomo/compatibility/overview]] — 兼容性总览