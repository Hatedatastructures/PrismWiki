---
title: "mihomo TUN 模式"
created: 2026-05-13
updated: 2026-05-13
type: client
tags: [tun, mihomo, client, network, routing]
related:
  - "[[client/mihomo-meta]]"
  - "[[client/mihomo-clash-config]]"
  - "[[client/mihomo-dns]]"
  - "[[dev/tcp]]"
  - "[[dev/udp]]"
  - "[[agent]]"
---

# mihomo TUN 模式

TUN 模式是实现代理透明化的核心手段。通过创建虚拟网卡并劫持系统路由，mihomo 可以代理所有应用程序的网络流量，无需应用本身支持代理协议。

## 为什么需要 TUN 模式

传统的代理方式有两种局限：

| 方式 | 原理 | 局限 |
|------|------|------|
| 系统代理 | 设置 HTTP(S)_PROXY 环境变量 | 仅代理支持系统代理的应用 |
| 手动代理 | 应用内配置 SOCKS5/HTTP 代理 | 需逐个应用配置 |

TUN 模式在网络层工作，所有 TCP/UDP 流量都经过虚拟网卡，由 mihomo 处理后转发，实现真正的全局代理。

## TUN 配置

```yaml
tun:
  enable: true
  stack: gvisor           # 网络栈实现
  dns-hijack:
    - any:53              # 劫持所有 DNS 查询
  auto-route: true        # 自动添加路由
  auto-detect-interface: true  # 自动检测出站接口
  mtu: 9000
  strict-route: false
  route-address:
    - 0.0.0.0/0           # 代理所有 IPv4
    - "::/0"              # 代理所有 IPv6
```

## Stack 选项

mihomo 提供三种网络栈实现：

### gvisor（推荐）

- 用户态网络栈，基于 gVisor 实现
- 不依赖系统内核，兼容性最好
- 性能略低于 system，但足够日常使用
- 适合大多数场景

### system

- 使用系统内核的网络栈
- 性能最好（内核态处理）
- 需要系统权限，部分平台可能受限
- Linux 上需要 `NET_ADMIN` 能力

### mixed

- TCP 使用 system，UDP 使用 gvisor
- 兼顾 TCP 性能和 UDP 兼容性
- 特定场景下的一种折中方案

## 自动路由（auto-route）

当 `auto-route: true` 时，mihomo 会：

1. 创建 TUN 虚拟网卡（Windows 上为 `Clash` 适配器）
2. 添加路由规则，将目标流量导入 TUN 接口
3. 设置 TUN 接口的 DNS 为 mihomo 监听地址

路由表变化示例（Windows）：

```
0.0.0.0/0 → TUN 接口（替代原默认路由）
128.0.0.0/1 → TUN 接口（拆分为两条，保留原默认路由）
```

这种拆分方式（"0.0.0.0/0 = 0/1 + 128/1"）确保 mihomo 的出站连接不受影响。

## DNS 劫持

TUN 模式下 DNS 劫持是关键环节：

```yaml
tun:
  dns-hijack:
    - any:53       # 劫持所有发往 53 端口的 DNS
    - tcp://any:53 # 同时劫持 TCP DNS
```

被劫持的 DNS 查询交给 mihomo 的 DNS 模块处理，配合 fake-ip 模式（见 [[client/mihomo-dns]]）实现完整的 DNS 处理链。

## 平台差异

### Windows

- 使用 WinTUN 驱动（`wintun.dll`）
- 需要管理员权限
- 自动创建 `Clash` 网络适配器
- 支持 `auto-route` 和 `auto-detect-interface`

### macOS

- 使用 utun 接口
- 需要 root 权限或 Network Extension
- `auto-route` 通过 `route` 命令添加路由
- 注意系统完整性保护（SIP）的影响

### Linux

- 使用 `/dev/net/tun`
- 需要 `CAP_NET_ADMIN` 能力
- systemd 下可用 `AmbientCapabilities=CAP_NET_ADMIN`
- 路由通过 `ip route` 管理

## TUN vs 代理模式对比

| 特性 | TUN | 系统代理 | 手动代理 |
|------|-----|----------|----------|
| 覆盖范围 | 全部应用 | 支持系统代理的应用 | 单个应用 |
| UDP 支持 | ✅ | ❌ | 取决于协议 |
| 配置复杂度 | 低（一次设置） | 中 | 高（逐个配置） |
| 性能开销 | 略高（虚拟网卡） | 低 | 最低 |
| 权限要求 | 管理员/root | 通常不需要 | 不需要 |

## 常见问题

### TUN 启动失败

- Windows：检查 WinTUN 驱动是否正确，是否以管理员运行
- Linux：检查 `/dev/net/tun` 是否存在，权限是否足够
- macOS：检查是否需要 root 权限

### 与其他 VPN 冲突

多个 TUN/VPN 同时运行可能导致路由冲突。解决方案：
- 设置 `strict-route: false`
- 使用 `route-address` 限定代理范围
- 调整 TUN 接口的 metric 值

### 游戏/特殊应用

部分游戏使用自定义网络协议或检测虚拟网卡。可尝试：
- 切换 stack 模式（gvisor ↔ system）
- 针对特定进程/端口排除代理

## 最佳实践

1. 日常使用 gvisor stack，兼容性最好
2. 配合 fake-ip DNS 模式防止泄露
3. 定期检查 `auto-route` 是否正确设置路由
4. 避免同时运行多个 VPN/TUN 应用

## 相关页面

- [[client/mihomo-meta]] — mihomo 元数据
- [[client/mihomo-clash-config]] — 配置文件详解
- [[client/mihomo-dns]] — DNS 处理机制
- [[dev/tcp]] — TCP 协议基础
- [[dev/udp]] — UDP 协议基础
- [[agent]] — Prism Agent 概览
