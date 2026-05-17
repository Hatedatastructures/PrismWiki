---
title: "Redir 监听器"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/listener/inbound/redir.go"
tags: [mihomo, listeners, redir, iptables, 透明代理, linux]
created: 2026-05-17
updated: 2026-05-17
related: [overview, tun, tproxy]
---

# Redir 监听器

**类别**: Mihomo 入站监听器

## 概述

Redir 监听器利用 Linux iptables 的 REDIR 功能实现透明代理。REDIR 将 TCP 连接透明重定向到代理端口，无需客户端配置。

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
  - name: redir-in
    type: redir
    listen: 0.0.0.0:7893
```

### 完整配置

```yaml
listeners:
  - name: redir-in
    type: redir
    listen: 0.0.0.0:7893
    sniffer:
      enable: true
```

## 配置参数

### name

```yaml
name: redir-in
```

监听器名称。

### type

```yaml
type: redir
```

固定值 `redir`，表示 Redir 监听器。

### listen

```yaml
listen: 0.0.0.0:7893
```

监听地址，REDIR 重定向的目标端口。

### sniffer

```yaml
sniffer:
  enable: true
```

启用入站流量嗅探。

## 工作原理

```
Redir 透明代理流程：
┌─────────────────────────────────────────────┐
│                                             │
│  应用程序 (连接 1.2.3.4:80)                  │
│      │                                      │
│      │ connect()                            │
│      ▼                                      │
│  Linux 网络栈                               │
│      │                                      │
│      │ iptables 规则                        │
│      │ REDIR → 127.0.0.1:7893              │
│      ▼                                      │
│  Mihomo (Redir 监听器)                      │
│      │                                      │
│      │ getsockopt(SO_ORIGINAL_DST)        │
│      │ 获取原始目标: 1.2.3.4:80            │
│      ▼                                      │
│  路由决策                                    │
│      │                                      │
│      ▼                                      │
│  出站代理                                    │
│                                             │
└─────────────────────────────────────────────┘
```

## iptables 配置

### 基础规则

```bash
# 创建链
iptables -t nat -N MIHOMO

# 排除本地流量
iptables -t nat -A MIHOMO -d 0.0.0.0/8 -j RETURN
iptables -t nat -A MIHOMO -d 10.0.0.0/8 -j RETURN
iptables -t nat -A MIHOMO -d 127.0.0.0/8 -j RETURN
iptables -t nat -A MIHOMO -d 169.254.0.0/16 -j RETURN
iptables -t nat -A MIHOMO -d 172.16.0.0/12 -j RETURN
iptables -t nat -A MIHOMO -d 192.168.0.0/16 -j RETURN
iptables -t nat -A MIHOMO -d 224.0.0.0/4 -j RETURN
iptables -t nat -A MIHOMO -d 240.0.0.0/4 -j RETURN

# 重定向到代理端口
iptables -t nat -A MIHOMO -p tcp -j REDIRECT --to-ports 7893

# 应用规则
iptables -t nat -A OUTPUT -p tcp -j MIHOMO
```

### 透明代理网关

```bash
# 启用 IP 转发
sysctl -w net.ipv4.ip_forward=1

# NAT 规则
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# PREROUTING 链（网关模式）
iptables -t nat -A PREROUTING -p tcp -j MIHOMO
```

### IPv6 支持

```bash
# IPv6 规则
ip6tables -t nat -N MIHOMO6
ip6tables -t nat -A MIHOMO6 -p tcp -j REDIRECT --to-ports 7893
ip6tables -t nat -A OUTPUT -p tcp -j MIHOMO6
```

## 配置示例

### 基础透明代理

```yaml
listeners:
  - name: redir-in
    type: redir
    listen: 0.0.0.0:7893
    sniffer:
      enable: true
```

### 网关透明代理

```yaml
listeners:
  - name: redir-in
    type: redir
    listen: 0.0.0.0:7893
    sniffer:
      enable: true
      sniff:
        HTTP:
          ports: [80, 8080-8090]
          override-destination: true
        TLS:
          ports: [443, 8443]
```

## 与 TProxy 的区别

| 特性 | Redir | TProxy |
|------|-------|--------|
| TCP | 支持 | 支持 |
| UDP | 不支持 | 支持 |
| 原始目标 | getsockopt | getsockname |
| 复杂度 | 简单 | 较复杂 |

建议：
- 仅需 TCP 透明代理时使用 Redir
- 需要 UDP 透明代理时使用 TProxy

## 排错指南

### 检查 iptables 规则

```bash
# 查看 NAT 规则
iptables -t nat -L -n -v

# 查看 MIHOMO 链
iptables -t nat -L MIHOMO -n -v
```

### 检查原始目标地址

```bash
# 使用 getsockopt 测试
# 原始目标存储在 SO_ORIGINAL_DST
```

### 连接问题排查

1. 检查 iptables 规则是否生效
2. 检查 Mihomo 监听端口
3. 检查路由规则配置
4. 检查日志输出

## 相关链接

- [[ref/mihomo/listeners/overview|Listeners 概览]] — 监听器总览
- [[ref/mihomo/listeners/tun|TUN 监听器]] — TUN 设备透明代理
- [[ref/mihomo/listeners/tproxy|TProxy 监听器]] — Linux iptables TPROXY