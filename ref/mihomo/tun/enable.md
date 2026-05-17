---
title: "TUN enable 配置"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/tun
source: "mihomo-Memo/transport/tun/tun.go"
tags: [mihomo, tun, enable, platform, driver]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/tun/overview]]"
  - "[[ref/mihomo/tun/stack]]"
  - "[[ref/mihomo/tun/auto-route]]"
---

# TUN enable 配置

**类别**: Mihomo TUN 配置 | **参数**: enable

## 配置格式

```yaml
tun:
  enable: true     # 启用 TUN 模式
  # enable: false  # 禁用 TUN 模式（默认）
```

## 参数说明

| 值 | 说明 |
|-----|------|
| true | 启用 TUN 虚拟网卡，开始透明代理 |
| false | 禁用 TUN 模式，不创建虚拟网卡 |

## 平台配置要求

### Windows

```yaml
tun:
  enable: true
  stack: gvisor
  auto-route: true
  auto-detect-interface: true
```

**依赖**:
- 需要 `wintun.dll` 文件（放在 mihomo 同目录）
- 需要管理员权限运行
- 会自动创建 "Clash" 网络适配器

### macOS

```yaml
tun:
  enable: true
  stack: gvisor
```

**依赖**:
- 使用系统原生 utun 驱动
- 可能需要 root 权限
- 注意 SIP (System Integrity Protection) 限制

### Linux

```yaml
tun:
  enable: true
  stack: system    # 推荐 system stack
```

**依赖**:
- `/dev/net/tun` 设备必须存在
- 需要 CAP_NET_ADMIN 能力
- systemd 配置示例:

```ini
[Service]
AmbientCapabilities=CAP_NET_ADMIN
```

### Android/iOS

移动端通过系统 API 实现:
- Android: VpnService API
- iOS: Network Extension framework

## 启用流程

```
enable: true
    │
    ▼
┌─────────────────────────────────────┐
│  检查平台权限                        │
│  Windows: 管理员                    │
│  Linux: CAP_NET_ADMIN               │
│  macOS: root                        │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  创建 TUN 设备                       │
│  Windows: WinTUN                    │
│  macOS: utun                        │
│  Linux: /dev/net/tun                │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  配置 Stack                          │
│  gVisor / System / Mixed            │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  配置路由 (auto-route)               │
│  添加路由规则                        │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  开始处理流量                        │
└─────────────────────────────────────┘
```

## 常见问题

### 启动失败

**Windows**:
```powershell
# 检查 wintun.dll
dir wintun.dll

# 检查管理员权限
# 需要以管理员身份运行
```

**Linux**:
```bash
# 检查 TUN 设备
ls -la /dev/net/tun

# 如果不存在，创建设备
mkdir -p /dev/net && mknod /dev/net/tun c 10 200

# 检查权限
chmod 666 /dev/net/tun
```

**macOS**:
```bash
# 检查 SIP 状态
csrutil status

# 可能需要特殊签名
```

## 相关链接

- [[overview]] — TUN 模式总览
- [[stack]] — 网络栈选择
- [[auto-route]] — 自动路由配置
- [[client/tun]] — 完整 TUN 说明