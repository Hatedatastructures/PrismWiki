---
title: "UDP Relay 实现"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/implementation
source: "mihomo-Meta/adapter/outbound/"
tags: [mihomo, udp, relay, fullcone, nat, implementation]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/implementation/overview]]"
  - "[[ref/network/udp]]"
  - "[[ref/mihomo/compatibility/features]]"
---

# UDP Relay 实现

**类别**: Mihomo 实现机制 | **模块**: UDP 中继详解

## 概述

UDP 中继是代理系统中较复杂的功能，涉及 NAT 映射、超时管理、数据转发等。mihomo 支持多种 UDP 中继模式。

## UDP 中继模式

### Native 模式

Native 模式是最基础的 UDP 中继:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Native UDP Relay                              │
└─────────────────────────────────────────────────────────────────────┘

客户端发送 UDP 数据包
    │
    ▼
┌─────────────────────┐
│  mihomo 接收 UDP    │
│  建立 UDP socket    │
│  发送到代理服务器    │
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│  代理服务器转发     │
│  UDP 数据到目标     │
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│  响应返回          │
│  NAT 映射还原       │
│  返回客户端         │
└─────────────────────┘
```

特点:
- 简单直接
- 无额外开销
- 但可能有 NAT 映射问题

### Fullcone NAT 模式

Fullcone NAT 提供更完整的 UDP 支持:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Fullcone NAT UDP                              │
└─────────────────────────────────────────────────────────────────────┘

特点：
┌─────────────────────────────────────────────────────────────────────┐
│  - 任意 IP:Port 可向映射后的地址发送数据                            │
│  - 类似开放端口效果                                                 │
│  - 对 P2P 应用友好                                                  │
│  - 需要代理服务器支持                                               │
└─────────────────────────────────────────────────────────────────────┘

工作流程：
客户端 A (192.168.1.1:1000)
    │
    ▼
┌─────────────────────┐
│  NAT 映射           │
│  内部: 外部          │
│  192.168.1.1:1000   │
│  → 代理IP:5000      │
└─────────────────────┘
    │
    ▼
任何客户端可向代理IP:5000 发送数据
    │
    ▼
┌─────────────────────┐
│  数据转发           │
│  无需验证源地址     │
│  返回客户端 A       │
└─────────────────────┘
```

## 协议 UDP 支持

### Trojan UDP

```yaml
proxies:
  - name: "trojan-udp"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    udp: true           # 启用 UDP
```

Trojan UDP 特点:
- 通过 TLS 连接复用
- UDP 数据封装在 TCP 流中
- 支持 Fullcone NAT

### VLESS UDP

```yaml
proxies:
  - name: "vless-udp"
    type: vless
    server: server.com
    port: 443
    uuid: "uuid"
    udp: true           # 启用 UDP
```

VLESS UDP 特点:
- 原生 UDP 支持
- XTLS 模式支持 UDP
- 支持 Fullcone NAT

### Shadowsocks UDP

```yaml
proxies:
  - name: "ss-udp"
    type: ss
    server: server.com
    port: 8388
    cipher: aes-128-gcm
    password: "password"
    udp: true           # 启用 UDP
    udp-over-tcp: false # UDP over TCP 模式
```

Shadowsocks UDP 特点:
- 原生 UDP 加密
- 支持 Fullcone NAT
- UDP over TCP 可选

### Hysteria2 UDP

```yaml
proxies:
  - name: "hysteria2-udp"
    type: hysteria2
    server: server.com
    port: 443
    password: "password"
    # QUIC 内置，UDP 自然支持
    up: "100 Mbps"
    down: "200 Mbps"
```

Hysteria2 UDP 特点:
- 基于 QUIC，原生 UDP
- 高性能 UDP 中继
- 拥塞控制优化

### TUIC UDP

```yaml
proxies:
  - name: "tuic-udp"
    type: tuic
    server: server.com
    port: 443
    uuid: "uuid"
    password: "password"
    udp-relay-mode: native  # native/quic
```

TUIC UDP 模式:
- `native`: 原生 UDP
- `quic`: UDP over QUIC stream

## UDP Over TCP

某些场景需要 UDP over TCP:

```yaml
proxies:
  - name: "ss-udp-over-tcp"
    type: ss
    server: server.com
    port: 8388
    cipher: aes-128-gcm
    password: "password"
    udp: true
    udp-over-tcp: true      # UDP 封装为 TCP
    udp-over-tcp-version: 2 # 版本
```

UDP over TCP 适用场景:
- UDP 不稳定的环境
- 防火墙限制 UDP
- TCP 性能更好

## NAT 映射管理

### 映射超时

UDP NAT 映射需要超时管理:

```yaml
# UDP NAT 超时配置（内部参数）
# 默认约 5 分钟无活动则清理
```

超时机制:
- 活动时更新超时时间
- 无活动则清理映射
- 防止映射表膨胀

### 映射表结构

```
┌─────────────────────────────────────┐
│          UDP NAT 映射表              │
├─────────────────────────────────────┤
│                                      │
│  内部地址        外部映射             │
│  ─────────────   ─────────────      │
│  192.168.1.1:1000 → 代理:5000       │
│  192.168.1.2:2000 → 代理:5001       │
│  ...                                │
│                                      │
│  映射数量: N                         │
│  超时时间: 300s                      │
│                                      │
└─────────────────────────────────────┘
```

## UDP 性能优化

### Hysteria2 UDP 优化

```yaml
proxies:
  - name: "hysteria2-optimized"
    type: hysteria2
    server: server.com
    port: 443
    password: "password"
    up: "500 Mbps"      # 高带宽
    down: "500 Mbps"
    # QUIC 拥塞控制优化 UDP
```

### 多路复用 UDP

```yaml
proxies:
  - name: "mux-udp"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    udp: true
    smux:
      enabled: true
      protocol: h2mux
      only-tcp: false   # UDP 也走 Mux
```

## UDP 问题排查

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| UDP 不通 | 协议不支持 UDP | 检查协议配置 |
| UDP 延迟高 | Fullcone 未启用 | 检查 Fullcone 支持 |
| UDP 丢包 | MTU 问题 | 检查 MTU 设置 |
| NAT 映射失效 | 超时清理 | 检查超时设置 |

### UDP 调试

```bash
# 测试 UDP 连接
nslookup google.com  # DNS 是 UDP

# 测试 QUIC 协议
curl --http3 https://cloudflare.com

# 查看 UDP 连接状态
netstat -an | grep udp
```

## 源码位置

- UDP 适配器: `adapter/outbound/*.go`
- NAT 管理: 内部实现
- UDP over TCP: `transport/udpotcp/`

## 相关链接

- [[overview]] — 实现总览
- [[ref/network/udp]] — UDP 协议基础
- [[ref/mihomo/compatibility/features]] — 功能支持
- [[ref/mihomo/compatibility/protocol-matrix]] — 协议支持矩阵