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
- [[ref/mihomo/tun/auto-route|auto-route]] — 自动路由配置
- [[ref/mihomo/tun/overview|tun]] — 完整 TUN 说明

## 接口检测原理

### 为什么需要自动检测

在 TUN 模式下，mihomo 需要将处理后的流量通过正确的物理网卡发送出去。如果选错了网卡，流量无法到达代理服务器，代理就完全无法工作。

```
正确场景:
应用 → TUN → mihomo → [eth0] → 互联网 → 代理服务器
                        ↑
                   正确的出站网卡

错误场景:
应用 → TUN → mihomo → [wlan0] → 互联网 → ❌ 无法到达代理服务器
                        ↑
                   错误的出站网卡
```

### 检测原理

mihomo 的接口检测通过以下步骤确定最佳出站网卡：

**步骤 1: 枚举所有网络接口**

```go
// 伪代码
interfaces, _ := net.Interfaces()
for _, iface := range interfaces {
    if iface.Flags&net.FlagUp != 0 && iface.Flags&net.FlagLoopback == 0 {
        // 活动状态且非回环的接口
        candidates = append(candidates, iface)
    }
}
```

**步骤 2: 查找默认路由**

系统的路由表决定了默认网关，而默认网关绑定在特定网卡上：

```
Windows 路由表:
─────────────────────────────────────────
网络目标      网关           接口          跃点数
0.0.0.0       192.168.1.1   192.168.1.100  25
                                    ↑
                           这就是出站 IP
                           对应接口: Ethernet

Linux 路由表:
─────────────────────────────────────────
default via 192.168.1.1 dev eth0 proto dhcp
                                ↑
                           出站接口: eth0
```

**步骤 3: 排除虚拟接口**

需要排除的接口类型：

| 接口名称模式 | 类型 | 原因 |
|-------------|------|------|
| `tun*`, `utun*` | TUN/TAP | TUN 自身接口 |
| `docker*`, `br-*` | Docker 网桥 | 容器网络 |
| `veth*`, `virbr*` | 虚拟网卡 | 虚拟机网络 |
| `vmnet*` | VMware | 虚拟化网络 |
| `*VPN*`, `ppp*` | VPN | 可能导致路由冲突 |
| `Loopback Pseudo` | 回环 | 本地回环 |

**步骤 4: 选择最佳接口**

如果存在多个候选接口（如同时有有线和无线），按优先级选择：

```
优先级排序:
1. 有线网卡（Ethernet/eth*）→ 最稳定
2. 无线网卡（Wi-Fi/wlan*）→ 次选
3. 其他物理网卡 → 最后选择
```

## 多网卡环境处理

### 典型多网卡场景

```
┌─────────────────────────────────────────────┐
│          多网卡环境示例                        │
├─────────────────────────────────────────────┤
│                                              │
│  eth0 (有线)     192.168.1.100/24            │
│    └─ 默认网关: 192.168.1.1                  │
│    └─ 状态: UP                               │
│                                              │
│  wlan0 (无线)    192.168.2.100/24            │
│    └─ 状态: UP（但非默认路由）                 │
│                                              │
│  docker0 (虚拟)  172.17.0.1/16               │
│    └─ 状态: UP                               │
│    └─ 排除（虚拟接口）                        │
│                                              │
│  tun0 (TUN)      198.18.0.1/16               │
│    └─ 状态: UP                               │
│    └─ 排除（TUN 接口自身）                    │
│                                              │
│  ✅ 选择结果: eth0（默认路由所在接口）          │
│                                              │
└─────────────────────────────────────────────┘
```

### 双默认路由冲突

某些情况下系统可能存在多个默认路由：

```bash
# Linux 示例
default via 192.168.1.1 dev eth0 metric 100
default via 192.168.2.1 dev wlan0 metric 200
```

mihomo 会选择**metric 值最小**的默认路由对应的接口（metric 越小优先级越高）。

### VPN 共存问题

当系统中存在 VPN 连接时，接口检测可能选错网卡：

```yaml
# 方案 1: 关闭自动检测，手动指定
tun:
  auto-detect-interface: false
  # interface-name: eth0

# 方案 2: 使用 strict-route
tun:
  strict-route: true       # 更严格的路由控制
  auto-detect-interface: true
```

### 接口热切换

如果网络接口在运行过程中发生变化（如拔掉网线切换到 Wi-Fi）：

| 行为 | 说明 |
|------|------|
| 当前连接中断 | mihomo 需要重新连接代理 |
| 接口名称变化 | 需要重新检测接口 |
| 解决方案 | 重启 TUN 模式或重新加载配置 |

## 默认路由选择

### 路由选择算法

mihomo 的接口检测本质上是一个"找到默认路由绑定的物理接口"的问题：

```
路由选择决策树:
                    遍历所有路由条目
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
         目标是 0.0.0.0/0?      不是默认路由
              │                    跳过
      ┌───────┴───────┐
      ▼               ▼
   有网关?          直连路由
      │              记录接口
      ▼
  网关对应的接口
      │
      ▼
  ┌────────────────────┐
  │ 排除虚拟/回环接口   │
  └────────┬───────────┘
           │
           ▼
  ┌────────────────────┐
  │ 按 metric 排序     │
  └────────┬───────────┘
           │
           ▼
  ┌────────────────────┐
  │ 选择 metric 最小的  │
  └────────────────────┘
```

### 平台检测命令

#### Windows

```powershell
# 查看带 metric 的路由表
Get-NetRoute -DestinationPrefix "0.0.0.0/0" | Sort-Object RouteMetric

# 输出示例:
# DestinationPrefix  : 0.0.0.0/0
# NextHop            : 192.168.1.1
# InterfaceAlias     : Ethernet      ← 这就是出站接口
# InterfaceIndex     : 12
# RouteMetric        : 25            ← metric 最小
```

#### Linux

```bash
# 查看默认路由（按 metric 排序）
ip route show default

# 输出示例:
# default via 192.168.1.1 dev eth0 proto dhcp metric 100
# default via 192.168.2.1 dev wlan0 proto dhcp metric 200
#                                     ↑ 选 eth0 (metric 100)
```

#### macOS

```bash
# 查看默认路由
netstat -rn | grep default

# 或使用 networksetup
networksetup -listnetworkserviceorder | grep "Hardware Port"
```

### 配置验证

配置完成后，可以验证 mihomo 是否选择了正确的接口：

```bash
# 查看 mihomo 日志
# 日志中会显示检测到的接口名称

# 手动验证接口是否正确
curl --interface eth0 https://api.ip.sb/ip  # 使用 eth0 查询 IP
curl --interface wlan0 https://api.ip.sb/ip  # 使用 wlan0 查询 IP
# 如果代理服务器只能从 eth0 访问，则 eth0 是正确的
```