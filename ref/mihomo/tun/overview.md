---
title: "mihomo TUN 模式参考"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/tun
source: "mihomo-Memo/transport/tun/"
tags: [mihomo, tun, network, routing, transparent-proxy]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/tun/enable]]"
  - "[[ref/mihomo/tun/stack]]"
  - "[[ref/mihomo/tun/dns-hijack]]"
  - "[[ref/mihomo/tun/auto-route]]"
  - "[[ref/mihomo/tun/auto-detect-interface]]"
  - "[[ref/mihomo/tun/overview|tun]]"
  - "[[ref/network/tcp]]"
  - "[[ref/network/udp]]"
---

# mihomo TUN 模式参考

**类别**: Mihomo 配置参考 | **模块**: TUN 透明代理

## 概述

TUN 模式是 mihomo 实现透明代理的核心功能，通过虚拟网络设备在系统网络层捕获和注入 IP 数据包，实现全局流量代理。

### 核心特性

| 特性 | 描述 |
|------|------|
| 透明代理 | 无需应用配合，自动代理所有流量 |
| UDP 支持 | 完整支持 UDP 协议，包括 DNS、游戏等 |
| DNS 劫持 | 内置 DNS 劫持功能，防止 DNS 泄露 |
| 进程识别 | 支持进程级别规则分流 |
| 跨平台 | Windows/macOS/Linux/Android/iOS |

### 工作原理

```
应用流量 → 系统路由 → TUN 设备 → mihomo 处理 → 代理/直连
           ↑                                    ↓
           └──────────── 响应注入 ←─────────────┘
```

## 配置结构

```yaml
tun:
  enable: true                    # 启用 TUN
  stack: gvisor                   # 网络栈实现
  dns-hijack:                     # DNS 劫持
    - any:53
    - tcp://any:53
  auto-route: true                # 自动路由
  auto-detect-interface: true     # 自动检测接口
  mtu: 9000                       # MTU 设置
  strict-route: false             # 严格路由
  route-address:                  # 代理地址范围
    - 0.0.0.0/0
    - "::/0"
  route-exclude-address:          # 排除地址
    - 192.168.0.0/16
```

## 子页面导航

| 页面 | 描述 |
|------|------|
| [[enable]] | TUN 启用与平台配置 |
| [[stack]] | 网络栈实现选择 (gVisor/System/Mixed) |
| [[dns-hijack]] | DNS 劫持配置详解 |
| [[auto-route]] | 自动路由机制 |
| [[auto-detect-interface]] | 自动接口检测 |

## 平台差异

| 平台 | 驱动 | 权限要求 |
|------|------|----------|
| Windows | WinTUN | 管理员 |
| macOS | utun | root |
| Linux | /dev/net/tun | CAP_NET_ADMIN |
| Android | VPN API | VPN 权限 |
| iOS | Network Extension | 特殊签名 |

## 源码位置

- **核心实现**: `transport/tun/`
- **Stack 实现**: `transport/tun/gvisor/`, `transport/tun/system/`
- **路由管理**: `transport/tun/route.go`

## 相关链接

- [[ref/mihomo/tun/overview|tun]] — TUN 模式完整说明
- [[ref/mihomo/config/overview]] — 配置总览
- [[ref/network/tcp]] — TCP 协议基础
- [[ref/network/udp]] — UDP 协议基础
- [[ref/mihomo/dns/overview]] — DNS 配置参考