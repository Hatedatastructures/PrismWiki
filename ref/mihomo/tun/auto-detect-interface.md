---
title: "TUN auto-detect-interface 配置"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/tun
source: "mihomo-Memo/transport/tun/"
tags: [mihomo, tun, interface, detect, gateway]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/tun/overview]]"
  - "[[ref/mihomo/tun/auto-route]]"
---

# TUN auto-detect-interface 配置

**类别**: Mihomo TUN 配置 | **参数**: auto-detect-interface

## 配置格式

```yaml
tun:
  enable: true
  auto-route: true
  auto-detect-interface: true   # 自动检测出站接口
```

## 参数说明

| 值 | 说明 |
|-----|------|
| true | 自动检测物理网卡，用于 mihomo 出站 |
| false | 使用系统默认接口 |

## 工作原理

```
auto-detect-interface: true
    │
    ▼
┌─────────────────────────────────────┐
│  1. 检查系统网络接口列表              │
│     列出所有活动网络接口              │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  2. 查找默认网关接口                  │
│     检查默认路由                      │
│     找出承载默认网关的接口            │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  3. 排除 TUN 接口                    │
│     排除 Clash/utun/tun 接口         │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  4. 选择最佳物理接口                  │
│     Ethernet > Wi-Fi > 其他          │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  5. 配置 mihomo 出站使用该接口        │
└─────────────────────────────────────┘
```

## 接口选择优先级

| 接口类型 | 优先级 | 说明 |
|----------|--------|------|
| Ethernet | 高 | 有线网络，稳定 |
| Wi-Fi | 中 | 无线网络 |
| Cellular | 低 | 移动数据 |
| Virtual | 排除 | VPN/TUN 接口 |

## 配置示例

### 基础配置

```yaml
tun:
  enable: true
  stack: gvisor
  auto-route: true
  auto-detect-interface: true
```

### 指定接口（替代自动检测）

如果自动检测不准确，可以手动指定:

```yaml
tun:
  enable: true
  auto-route: true
  # 手动指定接口
  # interface-name: eth0        # Linux
  # interface-name: Ethernet    # Windows
  # interface-name: en0         # macOS
```

## 平台差异

### Windows

```yaml
tun:
  auto-detect-interface: true
```

检测接口名称可能为:
- "Ethernet" (有线)
- "Wi-Fi" (无线)
- "以太网" (中文系统)

### Linux

```yaml
tun:
  auto-detect-interface: true
```

检测接口名称通常为:
- eth0, enp0s3 (有线)
- wlan0, wlp2s0 (无线)

### macOS

```yaml
tun:
  auto-detect-interface: true
```

检测接口名称通常为:
- en0 (有线/无线)
- en1 (第二网卡)

## 接口检测验证

### Windows

```powershell
# 查看网络接口
Get-NetAdapter | Where-Object { $_.Status -eq "Up" }

# 查看默认路由
route print 0.0.0.0
```

### Linux

```bash
# 查看接口
ip link show

# 查看默认路由
ip route show default
```

### macOS

```bash
# 查看接口
ifconfig

# 查看默认路由
netstat -rn | grep default
```

## 常见问题

### 接口检测不准确

可能原因:
- 多网卡环境
- VPN 同时运行
- 接口名称不规范

解决:
- 手动指定接口名称
- 使用 `route-exclude-address` 排除问题地址

### 与其他 VPN 冲突

```yaml
tun:
  auto-detect-interface: true
  strict-route: false    # 不强制路由优先
```

### 代理服务器连接问题

确保代理服务器 IP 不被导入 TUN:
- `route-exclude-address` 排除代理服务器 IP
- `auto-detect-interface` 正确检测出站接口

## 源码位置

- 接口检测: `transport/tun/interface.go`
- 平台适配: 各平台特定实现

## 相关链接

- [[overview]] — TUN 模式总览
- [[auto-route]] — 自动路由配置
- [[ref/mihomo/tun/overview|tun]] — 完整 TUN 说明