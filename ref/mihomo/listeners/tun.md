---
title: "TUN 监听器"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/listener/inbound/tun.go"
tags: [mihomo, listeners, tun, 透明代理, 系统代理]
created: 2026-05-17
updated: 2026-05-17
related: [overview, redir, tproxy]
---

# TUN 监听器

**类别**: Mihomo 入站监听器

## 概述

TUN 监听器通过创建虚拟网络接口（TUN 设备）实现透明代理。所有系统流量自动路由到 TUN 设备，由 Mihomo 处理，无需手动配置每个应用的代理设置。

## 基础配置

### 最小配置

```yaml
listeners:
  - name: tun-in
    type: tun
```

### 完整配置

```yaml
listeners:
  - name: tun-in
    type: tun
    stack: system
    dns-hijack:
      - any:53
    auto-route: true
    auto-detect-interface: true
```

## 配置参数

### name

```yaml
name: tun-in
```

监听器名称。

### type

```yaml
type: tun
```

固定值 `tun`，表示 TUN 监听器。

### stack

```yaml
stack: system
```

网络栈实现：

| 值 | 说明 | 性能 |
|-----|------|------|
| system | 系统网络栈 | 中等 |
| gvisor | gVisor 用户态栈 | 较低 |
| lwip | lwIP 用户态栈 | 较高（需 CGO） |

推荐：Linux 使用 `system`，Windows/macOS 使用 `gvisor` 或 `system`。

### dns-hijack

```yaml
dns-hijack:
  - any:53
  - tcp://any:53
```

DNS 劫持配置，将 DNS 请求转发到 Mihomo DNS 模块。

| 格式 | 说明 |
|------|------|
| any:53 | 劫持所有 53 端口 UDP DNS |
| tcp://any:53 | 劫持 TCP DNS |
| 8.8.8.8:53 | 劫持特定 IP 的 DNS |

### auto-route

```yaml
auto-route: true
```

自动配置路由表，将流量路由到 TUN 设备。

### auto-detect-interface

```yaml
auto-detect-interface: true
```

自动检测出站网络接口。

### endpoint-independent-nat

```yaml
endpoint-independent-nat: true
```

端点无关 NAT 模式，用于 UDP 代理。

| 值 | 说明 |
|-----|------|
| true | 端点无关 NAT（更兼容） |
| false | 端点相关 NAT（更安全） |

### mtu

```yaml
mtu: 9000
```

TUN 设备 MTU 值。默认 9000。

### inet4-address / inet6-address

```yaml
inet4-address: 172.19.0.1/30
inet6-address: fdfe:dcba:9876::1/126
```

TUN 设备 IP 地址。

### strict-route

```yaml
strict-route: true
```

严格路由模式，确保所有流量都经过 TUN。

## 平台支持

### Linux

```yaml
listeners:
  - name: tun-in
    type: tun
    stack: system
    auto-route: true
    auto-detect-interface: true
```

需要 root 权限或 `CAP_NET_ADMIN` 能力。

### Windows

```yaml
listeners:
  - name: tun-in
    type: tun
    stack: gvisor
    auto-route: true
    auto-detect-interface: true
```

需要管理员权限安装 Wintun 驱动。

### macOS

```yaml
listeners:
  - name: tun-in
    type: tun
    stack: system
    auto-route: true
    auto-detect-interface: true
```

需要 root 权限。

## TUN 工作原理

```
TUN 代理流程：
┌─────────────────────────────────────────────┐
│                                             │
│  应用程序                                    │
│      │                                      │
│      │ sendto()                             │
│      ▼                                      │
│  操作系统网络栈                              │
│      │                                      │
│      │ 路由表                               │
│      ▼                                      │
│  TUN 设备 (tun0)                            │
│      │                                      │
│      │ 读取                                 │
│      ▼                                      │
│  Mihomo (TUN 监听器)                        │
│      │                                      │
│      │ 路由决策                              │
│      ▼                                      │
│  出站代理                                    │
│                                             │
└─────────────────────────────────────────────┘
```

## 配置示例

### Linux 完整配置

```yaml
listeners:
  - name: tun-in
    type: tun
    stack: system
    dns-hijack:
      - any:53
    auto-route: true
    auto-detect-interface: true
    strict-route: true
    mtu: 9000
    inet4-address: 172.19.0.1/30
    inet6-address: fdfe:dcba:9876::1/126
```

### Windows 完整配置

```yaml
listeners:
  - name: tun-in
    type: tun
    stack: gvisor
    dns-hijack:
      - any:53
    auto-route: true
    auto-detect-interface: true
```

### macOS 完整配置

```yaml
listeners:
  - name: tun-in
    type: tun
    stack: system
    dns-hijack:
      - any:53
    auto-route: true
    auto-detect-interface: true
```

## 排错指南

### Linux 权限问题

```bash
# 使用 sudo 运行
sudo mihomo -d /path/to/config

# 或添加能力
sudo setcap cap_net_admin,cap_net_bind_service=+ep /usr/bin/mihomo
```

### Windows 驱动问题

确保安装了 Wintun 驱动：

1. 下载 Wintun 驱动
2. 解压到系统目录
3. 以管理员身份运行 Mihomo

### 路由冲突

检查路由表：

```bash
# Linux
ip route show

# macOS
netstat -rn
```

### DNS 泄漏

确保 DNS 劫持配置正确：

```yaml
dns-hijack:
  - any:53
  - tcp://any:53
```

## 相关链接

- [[ref/mihomo/listeners/overview|Listeners 概览]] — 监听器总览
- [[ref/mihomo/listeners/redir|Redir 监听器]] — Linux iptables REDIR
- [[ref/mihomo/listeners/tproxy|TProxy 监听器]] — Linux iptables TPROXY