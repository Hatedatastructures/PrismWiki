---
title: "TUN auto-route 配置"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/tun
source: "mihomo-Memo/transport/tun/route.go"
tags: [mihomo, tun, auto-route, routing, gateway]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/tun/overview]]"
  - "[[ref/mihomo/tun/auto-detect-interface]]"
---

# TUN auto-route 配置

**类别**: Mihomo TUN 配置 | **参数**: auto-route

## 配置格式

```yaml
tun:
  enable: true
  auto-route: true        # 自动配置系统路由
  # auto-route: false     # 手动配置路由
```

## 参数说明

| 值 | 说明 |
|-----|------|
| true | 自动添加路由规则，将流量导入 TUN |
| false | 不自动配置，需要手动设置路由 |

## 自动路由流程

```
auto-route: true
    │
    ▼
┌─────────────────────────────────────┐
│  1. 创建 TUN 接口                    │
│     Windows: Clash 适配器            │
│     macOS: utun0                    │
│     Linux: tun0                     │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  2. 配置接口 IP 地址                  │
│     TUN IP: 198.18.0.1              │
│     子网: 255.255.0.0               │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  3. 添加路由规则                      │
│     拆分路由方式                      │
│     0.0.0.0/1 → TUN                 │
│     128.0.0.0/1 → TUN               │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  4. IPv6 路由                        │
│     ::/1 → TUN                      │
│     8000::/1 → TUN                  │
└─────────────────────────────────────┘
```

## 路由拆分原理

mihomo 使用拆分路由而非覆盖默认路由:

```
┌─────────────────────────────────────┐
│          拆分路由原理                 │
├─────────────────────────────────────┤
│                                      │
│  原理：                              │
│  0.0.0.0/0 = 0.0.0.0/1 + 128.0.0.0/1│
│                                      │
│  ::/0 = ::/1 + 8000::/1             │
│                                      │
│  优势：                              │
│  - 不覆盖原默认路由                   │
│  - mihomo 出站流量不受影响            │
│  - 代理服务器 IP 可直连               │
│                                      │
└─────────────────────────────────────┘
```

### Windows 路由命令

```powershell
# mihomo 自动执行的等效命令
route add 0.0.0.0 mask 128.0.0.0 198.18.0.1 metric 5
route add 128.0.0.0 mask 128.0.0.0 198.18.0.1 metric 5
```

### Linux 路由命令

```bash
# mihomo 自动执行的等效命令
ip route add 0.0.0.0/1 dev tun0
ip route add 128.0.0.0/1 dev tun0
```

### macOS 路由命令

```bash
# mihomo 自动执行的等效命令
route add 0.0.0.0/1 198.18.0.1
route add 128.0.0.0/1 198.18.0.1
```

## route-address 配置

控制代理的地址范围:

```yaml
tun:
  auto-route: true
  route-address:
    - 0.0.0.0/0         # 代理所有 IPv4
    - "::/0"            # 代理所有 IPv6
```

### 部分代理示例

```yaml
tun:
  auto-route: true
  route-address:
    - 10.0.0.0/8        # 仅代理特定网段
    - 172.16.0.0/12
```

## route-exclude-address 配置

排除特定地址:

```yaml
tun:
  auto-route: true
  route-exclude-address:
    - 192.168.0.0/16    # 排除局域网
    - 10.0.0.0/8        # 排除私有网络
    - 127.0.0.0/8       # 排除本地回环
```

## 配置示例

### 基础配置

```yaml
tun:
  enable: true
  auto-route: true
  auto-detect-interface: true
```

### 完整配置

```yaml
tun:
  enable: true
  stack: gvisor
  auto-route: true
  auto-detect-interface: true
  
  route-address:
    - 0.0.0.0/0
    - "::/0"
  
  route-exclude-address:
    - 192.168.0.0/16
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 127.0.0.0/8
```

### 企业内网配置

```yaml
tun:
  enable: true
  auto-route: true
  
  route-exclude-address:
    # 企业内网
    - 10.0.0.0/8
    - 172.16.0.0/12
    # 特定服务器
    - 192.168.1.100/32
```

## 源码位置

- 路由配置: `transport/tun/route.go`
- 平台适配: `transport/tun/gvisor/`, `transport/tun/system/`

## 相关链接

- [[overview]] — TUN 模式总览
- [[auto-detect-interface]] — 自动接口检测
- [[ref/mihomo/tun/overview|tun]] — 完整 TUN 说明