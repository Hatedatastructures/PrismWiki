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
- [[ref/mihomo/tun/auto-detect-interface|auto-detect-interface]] — 自动接口检测
- [[ref/mihomo/tun/overview|tun]] — 完整 TUN 说明

## 自动路由机制详解

### 为什么需要路由注入

操作系统的路由表决定了每个数据包从哪个网络接口发出。默认情况下，所有"非本地"流量通过默认网关（通常是物理网卡）发出。

要实现透明代理，必须将流量"引导"到 TUN 虚拟网卡。这就需要通过修改系统路由表来实现。

```
修改前:
┌──────────────────────────────────────┐
│          系统路由表                    │
├──────────────────────────────────────┤
│  0.0.0.0/0 → 192.168.1.1 (物理网卡)   │
│  192.168.1.0/24 → 直连 (物理网卡)     │
│  127.0.0.0/8 → lo (本地回环)          │
└──────────────────────────────────────┘

修改后:
┌──────────────────────────────────────┐
│          系统路由表                    │
├──────────────────────────────────────┤
│  0.0.0.0/1 → TUN (代理流量)           │
│  128.0.0.0/1 → TUN (代理流量)         │
│  192.168.1.0/24 → 直连 (物理网卡)     │
│  127.0.0.0/8 → lo (本地回环)          │
└──────────────────────────────────────┘
```

### 拆分路由 vs 覆盖路由

**覆盖路由方案（不推荐）：**
```bash
# 直接修改默认路由 — 危险！
ip route replace default dev tun0
```
问题：这会导致 mihomo 自身的出站流量也被路由到 TUN，形成死循环。

**拆分路由方案（mihomo 采用）：**
```bash
# 将 0.0.0.0/0 拆分为两个 /1 路由
ip route add 0.0.0.0/1 dev tun0     # 覆盖 0.0.0.0 - 127.255.255.255
ip route add 128.0.0.0/1 dev tun0   # 覆盖 128.0.0.0 - 255.255.255.255
```

**为什么拆分有效？**

路由匹配遵循**最长前缀匹配**原则。`/1` 的路由比 `/0` 更具体，但比 `/0` 更不具体。因此：

- 普通流量匹配 `0.0.0.0/1` 或 `128.0.0.0/1` → 走 TUN
- 代理服务器的具体 IP 路由（如 `/32`）仍然指向物理网卡 → mihomo 可以直连代理

```
路由匹配优先级（从高到低）:
/32 (代理服务器)  → 物理网卡（直连）
/24 (局域网)      → 物理网卡（直连）
...
/1 (拆分路由)     → TUN 设备
/0 (默认路由)     → 物理网卡（默认网关）
```

## 路由表注入原理

### 路由添加流程

```
mihomo 启动 TUN 模式
    │
    ▼
┌─────────────────────────────────────┐
│ 1. 保存现有路由表状态                 │
│    备份当前默认路由和关键路由          │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 2. 排除地址路由添加                   │
│    route-exclude-address 中的地址     │
│    添加到物理网卡路由                 │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 3. 拆分路由注入                       │
│    0.0.0.0/1 → TUN                   │
│    128.0.0.0/1 → TUN                 │
│    ::/1 → TUN (IPv6)                │
│    8000::/1 → TUN (IPv6)            │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 4. 验证路由表                         │
│    确认路由正确注入                   │
└─────────────────────────────────────┘
```

### 平台路由操作

#### Windows (route.exe)

```powershell
# 添加 IPv4 拆分路由
route add 0.0.0.0 mask 128.0.0.0 198.18.0.1 metric 5
route add 128.0.0.0 mask 128.0.0.0 198.18.0.1 metric 5

# 添加 IPv6 拆分路由
route add ::1 ::8000:0:0:0:0:0:0 198.18.0.1
route add 8000::1 8000:0:0:0:0:0:0 198.18.0.1

# 排除路由（局域网）
route add 192.168.0.0 mask 255.255.0.0 <物理网关> metric 1
```

**Metric 值含义：** Metric 越小优先级越高。拆分路由使用 metric 5 确保在直连路由之后、默认路由之前。

#### Linux (ip route)

```bash
# 创建独立路由表（避免影响全局路由）
ip rule add fwmark 1 table 100
ip route add local default dev tun0 table 100

# 或直接添加拆分路由
ip route add 0.0.0.0/1 dev tun0
ip route add 128.0.0.0/1 dev tun0

# IPv6
ip -6 route add ::/1 dev tun0
ip -6 route add 8000::/1 dev tun0

# 排除局域网
ip route add 192.168.0.0/16 dev eth0 proto static
```

#### macOS (route)

```bash
# 添加拆分路由
route add 0.0.0.0/1 198.18.0.1
route add 128.0.0.0/1 198.18.0.1

# IPv6
route add -inet6 ::/1 198.18.0.1
route add -inet6 8000::/1 198.18.0.1
```

## 防回环机制

### 什么是回环问题

回环（Loop）是 TUN 模式中最常见的故障模式：

```
危险场景:
mihomo → 连接代理服务器 → 数据包 → TUN → mihomo → 连接代理服务器 → ...
                              ▲                                        │
                              └──────────── 无限循环 ─────────────────────┘
```

如果代理服务器的流量被导入 TUN，而 TUN 又将流量送回 mihomo，就会形成无限循环。

### 防回环策略

mihomo 通过多层策略防止回环：

#### 1. 拆分路由（核心机制）

如前所述，拆分路由通过最长前缀匹配确保代理服务器的具体 IP 路由不受影响：

```
代理服务器 IP: 203.0.113.10/32  →  物理网卡（最长匹配）
普通目标 IP:    8.8.8.8         →  0.0.0.0/1 → TUN
```

#### 2. 路由排除（route-exclude-address）

显式排除特定地址段：

```yaml
tun:
  route-exclude-address:
    - 192.168.0.0/16      # 排除局域网
    - 10.0.0.0/8          # 排除私有网络
    - 172.16.0.0/12
    - 203.0.113.10/32     # 排除代理服务器
```

#### 3. 自动接口检测（auto-detect-interface）

确保 mihomo 使用正确的物理网卡连接代理服务器：

```yaml
tun:
  auto-detect-interface: true  # 自动选择出站网卡
```

#### 4. 标记/策略路由（Linux 专用）

Linux 可以使用 fwmark 标记 mihomo 的流量，然后通过策略路由绕过 TUN：

```bash
# 标记 mihomo 进程的流量
iptables -t mangle -A OUTPUT -m mark --mark 0x100 -j ACCEPT

# 策略路由：标记的流量不走 TUN
ip rule add fwmark 0x100 table main
```

#### 5. SO_MARK 套接字选项

mihomo 可以为自己的出站连接设置 `SO_MARK`，配合 iptables 规则绕过 TUN：

```bash
# 标记的流量不接受 TUN 路由
iptables -t mangle -A OUTPUT -m mark --mark 0x100 -j CONNMARK --set-mark 0x100
```

### 回环检测与诊断

如果怀疑发生了回环，可以通过以下方式诊断：

```bash
# 查看路由表
ip route show          # Linux
route print            # Windows
netstat -rn            # macOS

# 查看 TUN 流量
tcpdump -i tun0        # 检查是否有异常的重复流量

# 查看 mihomo 日志
# 回环通常会表现为连接超时或大量重复连接
```