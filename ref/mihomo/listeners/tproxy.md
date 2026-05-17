---
title: "TProxy 监听器"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/listener/inbound/tproxy.go"
tags: [mihomo, listeners, tproxy, iptables, 透明代理, linux, udp]
created: 2026-05-17
updated: 2026-05-17
related: [overview, tun, redir]
---

# TProxy 监听器

**类别**: Mihomo 入站监听器

## 概述

TProxy 监听器利用 Linux iptables 的 TPROXY 功能实现透明代理。与 Redir 不同，TProxy 同时支持 TCP 和 UDP，是实现完整透明代理的推荐方案。

## 平台支持

| 平台 | 支持 |
|------|------|
| Linux | 支持 |
| macOS | 不支持 |
| Windows | 不支持 |

## 基础配置

### 最小配置

```yaml
listeners:
  - name: tproxy-in
    type: tproxy
    listen: 0.0.0.0:7894
```

### 完整配置

```yaml
listeners:
  - name: tproxy-in
    type: tproxy
    listen: 0.0.0.0:7894
    udp: true
    sniffer:
      enable: true
```

## 配置参数

### name

```yaml
name: tproxy-in
```

监听器名称。

### type

```yaml
type: tproxy
```

固定值 `tproxy`，表示 TProxy 监听器。

### listen

```yaml
listen: 0.0.0.0:7894
```

监听地址，TPROXY 重定向的目标端口。

### udp

```yaml
udp: true
```

是否启用 UDP 支持。

### sniffer

```yaml
sniffer:
  enable: true
```

启用入站流量嗅探。

## 工作原理

```
TProxy 透明代理流程：
┌─────────────────────────────────────────────┐
│                                             │
│  应用程序 (TCP/UDP)                          │
│      │                                      │
│      │ connect() / sendto()                │
│      ▼                                      │
│  Linux 网络栈                               │
│      │                                      │
│      │ iptables TPROXY 规则                 │
│      │                                     │
│      ├── TCP → TPROXY 127.0.0.1:7894      │
│      │                                     │
│      └── UDP → TPROXY 127.0.0.1:7894      │
│      │                                     │
│      ▼                                      │
│  Mihomo (TProxy 监听器)                     │
│      │                                      │
│      │ getsockname() 获取原始目标          │
│      ▼                                      │
│  路由决策                                    │
│      │                                      │
│      ▼                                      │
│  出站代理                                    │
│                                             │
└─────────────────────────────────────────────┘
```

## iptables 配置

### TCP TProxy 规则

```bash
# 创建链
iptables -t mangle -N MIHOMO_TCP

# 排除本地流量
iptables -t mangle -A MIHOMO_TCP -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A MIHOMO_TCP -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A MIHOMO_TCP -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A MIHOMO_TCP -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A MIHOMO_TCP -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A MIHOMO_TCP -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A MIHOMO_TCP -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A MIHOMO_TCP -d 240.0.0.0/4 -j RETURN

# TPROXY 重定向
iptables -t mangle -A MIHOMO_TCP -p tcp -j TPROXY --on-port 7894 --tproxy-mark 1

# 应用规则
iptables -t mangle -A PREROUTING -p tcp -j MIHOMO_TCP
iptables -t mangle -A OUTPUT -p tcp -j MIHOMO_TCP
```

### UDP TProxy 规则

```bash
# 创建链
iptables -t mangle -N MIHOMO_UDP

# 排除本地流量（同 TCP）

# TPROXY 重定向
iptables -t mangle -A MIHOMO_UDP -p udp -j TPROXY --on-port 7894 --tproxy-mark 1

# 应用规则
iptables -t mangle -A PREROUTING -p udp -j MIHOMO_UDP
```

### 策略路由

```bash
# 添加策略路由规则
ip rule add fwmark 1 lookup 100

# 添加路由表
ip route add local 0.0.0.0/0 dev lo table 100
```

### IPv6 支持

```bash
# IPv6 TCP
ip6tables -t mangle -N MIHOMO6_TCP
ip6tables -t mangle -A MIHOMO6_TCP -p tcp -j TPROXY --on-port 7894 --tproxy-mark 1
ip6tables -t mangle -A PREROUTING -p tcp -j MIHOMO6_TCP

# IPv6 UDP
ip6tables -t mangle -N MIHOMO6_UDP
ip6tables -t mangle -A MIHOMO6_UDP -p udp -j TPROXY --on-port 7894 --tproxy-mark 1
ip6tables -t mangle -A PREROUTING -p udp -j MIHOMO6_UDP

# IPv6 策略路由
ip -6 rule add fwmark 1 lookup 100
ip -6 route add local ::/0 dev lo table 100
```

## 配置示例

### 基础 TProxy

```yaml
listeners:
  - name: tproxy-in
    type: tproxy
    listen: 0.0.0.0:7894
```

### TCP + UDP TProxy

```yaml
listeners:
  - name: tproxy-in
    type: tproxy
    listen: 0.0.0.0:7894
    udp: true
    sniffer:
      enable: true
      sniff:
        HTTP:
          ports: [80, 8080-8090]
          override-destination: true
        TLS:
          ports: [443, 8443]
        QUIC:
          ports: [443, 8443]
```

### 网关透明代理

```yaml
listeners:
  - name: tproxy-in
    type: tproxy
    listen: 0.0.0.0:7894
    udp: true
    sniffer:
      enable: true
      parse-pure-ip: true
      sniff:
        HTTP:
          ports: [80, 8080-8090]
          override-destination: true
        TLS:
          ports: [443, 8443]
        QUIC:
          ports: [443, 8443]
```

## 与 Redir 的对比

| 特性 | TProxy | Redir |
|------|--------|-------|
| TCP | 支持 | 支持 |
| UDP | 支持 | 不支持 |
| 原始目标 | getsockname | getsockopt |
| 配置复杂度 | 中等 | 简单 |
| iptables 表 | mangle | nat |

建议：
- 需要 UDP 透明代理时使用 TProxy
- 仅 TCP 时可使用 Redir（配置更简单）

## 排错指南

### 检查 iptables 规则

```bash
# 查看 mangle 规则
iptables -t mangle -L -n -v

# 查看 TProxy 链
iptables -t mangle -L MIHOMO_TCP -n -v
iptables -t mangle -L MIHOMO_UDP -n -v
```

### 检查策略路由

```bash
# 查看路由规则
ip rule list

# 查看路由表
ip route show table 100
```

### 检查内核参数

```bash
# 启用 IP 转发
sysctl net.ipv4.ip_forward
sysctl -w net.ipv4.ip_forward=1

# 启用 TProxy
sysctl net.ipv4.conf.all.route_localnet
sysctl -w net.ipv4.conf.all.route_localnet=1
```

## 相关链接

- [[ref/mihomo/listeners/overview|Listeners 概览]] — 监听器总览
- [[ref/mihomo/listeners/tun|TUN 监听器]] — TUN 设备透明代理
- [[ref/mihomo/listeners/redir|Redir 监听器]] — Linux iptables REDIR