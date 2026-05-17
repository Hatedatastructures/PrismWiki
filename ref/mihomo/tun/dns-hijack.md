---
title: "TUN dns-hijack 配置"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/tun
source: "mihomo-Memo/transport/tun/"
tags: [mihomo, tun, dns, hijack, dns-leak]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/tun/overview]]"
  - "[[ref/mihomo/dns/overview]]"
  - "[[ref/protocol/dns-over-udp]]"
---

# TUN dns-hijack 配置

**类别**: Mihomo TUN 配置 | **参数**: dns-hijack

## 配置格式

```yaml
tun:
  enable: true
  dns-hijack:
    - any:53              # 劫持所有 UDP 53
    - tcp://any:53        # 劫持所有 TCP 53
```

## 劫持类型

### UDP DNS 劫持

```yaml
dns-hijack:
  - any:53        # 劫持所有目标端口 53 的 UDP 流量
```

**说明**:
- 拦截所有向端口 53 发送的 UDP DNS 查询
- `any` 表示任意目标 IP
- 最常用的 DNS 劫持方式

### TCP DNS 劫持

```yaml
dns-hijack:
  - tcp://any:53  # 劫持所有目标端口 53 的 TCP 流量
```

**说明**:
- 拦截 TCP DNS 查询
- 某些应用使用 TCP DNS (如长查询)
- 配合 UDP 劫持使用

### 特定地址劫持

```yaml
dns-hijack:
  - 8.8.8.8:53            # 仅劫持 Google DNS
  - 1.1.1.1:53            # 仅劫持 Cloudflare DNS
  - tcp://8.8.8.8:53      # TCP DNS 特定地址
```

**说明**:
- 可以指定具体 DNS 服务器地址
- 更精细的控制

## 工作流程

```
应用发送 DNS 查询
    │
    ▼
┌─────────────────────────────────────┐
│  系统向 DNS 服务器发送查询            │
│  例如：向 8.8.8.8:53 发送 UDP        │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  TUN 接口拦截流量                    │
│  检测目标端口 53                     │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  匹配 dns-hijack 规则                 │
│  any:53 → 匹配                       │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  转交 mihomo DNS 模块                │
│  根据 DNS 配置处理查询               │
│  fake-ip 模式返回虚拟 IP            │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  响应通过 TUN 返回应用               │
└─────────────────────────────────────┘
```

## 配置示例

### 基础配置

```yaml
tun:
  enable: true
  dns-hijack:
    - any:53        # 仅 UDP DNS
```

### 完整配置

```yaml
tun:
  enable: true
  stack: gvisor
  dns-hijack:
    - any:53              # UDP DNS
    - tcp://any:53        # TCP DNS
```

### 精细控制

```yaml
tun:
  enable: true
  dns-hijack:
    - any:53              # 所有 DNS
    - tcp://any:53        # TCP DNS
  dns-hijack-out:         # 出站 DNS 劫持
    - any:53
```

## 与 DNS 配置配合

TUN DNS 劫持需要配合 DNS 配置:

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - https://dns.alidns.com/dns-query
  fallback:
    - https://dns.google/dns-query

tun:
  enable: true
  dns-hijack:
    - any:53
    - tcp://any:53
```

## 防止 DNS 泄露

DNS 劫持的核心目的是防止 DNS 泄露:

```
┌─────────────────────────────────────┐
│          DNS 泄露风险                │
├─────────────────────────────────────┤
│                                      │
│  无 DNS 劫持:                        │
│  应用 DNS → 系统 DNS → 运营商 DNS    │
│            ↓                         │
│        泄露访问意图                   │
│                                      │
│  有 DNS 劫持:                        │
│  应用 DNS → TUN → mihomo → 代理 DNS │
│            ↓                         │
│        安全，无泄露                   │
│                                      │
└─────────────────────────────────────┘
```

## 常见问题

### DNS 劫持不生效

检查:
- `tun.enable: true`
- DNS 模块启用 `dns.enable: true`
- 路由正确配置

### 部分应用 DNS 泄露

某些应用可能:
- 硬编码 DNS 服务器
- 使用非标准端口
- 使用 DoH/DoT 直连

解决:
- 检查应用 DNS 设置
- 确保 DoH 通过代理
- 使用 fake-ip 模式

## 源码位置

- DNS 劫持实现: `transport/tun/dns.go`
- DNS 模块: `component/dns/`

## 相关链接

- [[overview]] — TUN 模式总览
- [[ref/mihomo/dns/overview]] — DNS 配置参考
- [[ref/protocol/dns-over-udp]] — DNS over UDP
- [[ref/protocol/dns-over-tcp]] — DNS over TCP
- [[client/mihomo-dns]] — DNS 处理机制