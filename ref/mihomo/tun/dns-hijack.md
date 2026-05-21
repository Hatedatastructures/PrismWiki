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
- [[ref/mihomo/tun/overview|tun]] — DNS 处理机制

## DNS 劫持原理

### 什么是 DNS 泄露

在代理环境中，DNS 泄露是指 DNS 查询没有通过代理通道，而是直接发送到了运营商的 DNS 服务器。这会导致：

1. **隐私泄露**: 运营商可以看到你访问的域名
2. **代理绕过**: 即使流量走了代理，DNS 返回的真实 IP 可能导致直连
3. **GFW 污染**: 在某些地区，运营商会返回错误的 DNS 结果（DNS 污染）

```
┌─────────────────────────────────────────────────────┐
│                  DNS 泄露场景                         │
│                                                      │
│  应用发出 DNS 查询: example.com                      │
│                                                      │
│  ┌─ 正确路径（无泄露）─┐                              │
│  │  应用 → TUN → mihomo → 代理 DNS → 8.8.8.8        │
│  │  mihomo 可以看到查询，可以控制返回结果              │
│  └────────────────────────────────────────────       │
│                                                      │
│  ┌─ 泄露路径 ────────┐                                │
│  │  应用 → 系统 DNS → 运营商 DNS → 223.5.5.5        │
│  │  mihomo 无法控制，运营商知道你在访问什么            │
│  └──────────────────────────────────────────         │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### DNS 劫持如何工作

DNS 劫持的本质是**拦截和重定向**：

1. **拦截**: 在 TUN 层面拦截所有目标端口为 53 的数据包
2. **解析**: 解析 DNS 查询报文内容
3. **重定向**: 将查询转交给 mihomo 的 DNS 模块处理
4. **响应**: DNS 模块返回结果（真实 IP 或 Fake IP）
5. **回写**: 将响应数据写回 TUN 设备，传递给原始应用

## TUN DNS 劫持流程

### 完整数据流

```
应用发起 DNS 查询
    │
    ▼
┌──────────────────────────────────────────────┐
│ 系统网络栈                                     │
│ 构造 DNS 查询报文                              │
│  ┌──────────────────────────────────────┐    │
│  │  Header: ID=0x1234, RD=1            │    │
│  │  Question: example.com A IN          │    │
│  └──────────────────────────────────────┘    │
└────────────────┬─────────────────────────────┘
                 │ UDP, 目标: 8.8.8.8:53
                 ▼
┌──────────────────────────────────────────────┐
│ TUN 设备拦截                                   │
│ 匹配 dns-hijack 规则 (any:53)                 │
└────────────────┬─────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────┐
│ mihomo DNS 模块                                │
│                                              │
│  1. 解析 DNS 查询报文                          │
│  2. 查询域名规则（geoip/geosite 等）            │
│  3. 决定使用哪个 DNS 上游                       │
│     - 国内域名 → 国内 DNS                       │
│     - 国外域名 → 国外 DNS                       │
│     - Fake-IP 模式 → 直接返回虚拟 IP            │
│                                              │
└────────────────┬─────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────┐
│ DNS 响应构造                                   │
│                                              │
│  Fake-IP 模式:                                │
│    Answer: example.com → 198.18.12.34         │
│                                              │
│  正常解析模式:                                  │
│    Answer: example.com → 93.184.216.34        │
│                                              │
└────────────────┬─────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────┐
│ 响应通过 TUN 返回应用                            │
│ 应用收到 DNS 响应，发起连接                     │
└──────────────────────────────────────────────┘
```

### DNS 报文拦截细节

DNS 报文格式：

```
┌──────────────────────────────────────────────┐
│                DNS 报文结构                     │
├──────────────────────────────────────────────┤
│  Header (12 字节)                             │
│  ├── Transaction ID (2 字节)                  │
│  ├── Flags (2 字节)                           │
│  ├── Question Count (2 字节)                  │
│  ├── Answer Count (2 字节)                    │
│  ├── Authority Count (2 字节)                 │
│  └── Additional Count (2 字节)                │
├──────────────────────────────────────────────┤
│  Question Section                             │
│  ├── QNAME (可变，域名编码)                     │
│  ├── QTYPE (2 字节)                           │
│  └── QCLASS (2 字节)                          │
├──────────────────────────────────────────────┤
│  Answer Section (查询时为空)                   │
│  ...                                         │
└──────────────────────────────────────────────┘
```

mihomo 在 TUN 层拦截到 UDP 数据包后：
1. 验证目标端口是否为 53
2. 解析 UDP 载荷中的 DNS 报文
3. 提取 `QNAME`（查询的域名）
4. 根据域名匹配规则决定处理方式

## Fake-IP 模式

### 原理

Fake-IP 是 mihomo 特有的 DNS 响应模式。它不会真正去查询 DNS，而是直接返回一个虚假的 IP 地址。

```
┌──────────────────────────────────────────────┐
│           Fake-IP 工作流程                     │
│                                              │
│  应用: example.com 的 IP 是什么？              │
│    │                                          │
│    ▼                                          │
│  mihomo DNS: 给你 198.18.12.34（Fake-IP）      │
│    │                                          │
│    ▼                                          │
│  应用: 好，我去连接 198.18.12.34:443            │
│    │                                          │
│    ▼                                          │
│  mihomo 拦截连接请求                             │
│    │                                          │
│    ▼                                          │
│  mihomo 查找映射表:                              │
│    198.18.12.34 → example.com                 │
│    │                                          │
│    ▼                                          │
│  mihomo 执行代理逻辑:                            │
│    域名: example.com                           │
│    规则: 匹配 GEOIP,CN → 直连                   │
│    或: 匹配国外 → 代理                          │
│    │                                          │
│    ▼                                          │
│  真实连接建立                                   │
└──────────────────────────────────────────────┘
```

### Fake-IP 的优势

| 优势 | 说明 |
|------|------|
| **零延迟** | 不需要等待 DNS 查询返回，立即响应 |
| **精准规则匹配** | 基于域名而非 IP 做规则判断，更准确 |
| **避免 DNS 污染** | 不依赖外部 DNS，不受污染影响 |
| **减少 DNS 查询** | 同一域名只查询一次（首次连接时） |

### Fake-IP 配置

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16     # Fake-IP 地址池
  fake-ip-filter:                  # 不使用 Fake-IP 的域名
    - "*.lan"
    - "*.local"
    - localhost.ptlogin2.qq.com

tun:
  enable: true
  dns-hijack:
    - any:53
```

### Fake-IP 地址范围

`198.18.0.0/15` 是 IANA 保留的基准测试地址范围（RFC 2544），不会在公网出现，因此非常适合用作 Fake-IP。

```
198.18.0.0/16 → 第一个 /16（mihomo 常用）
198.19.0.0/16 → 第二个 /16

mihomo 默认使用 198.18.0.0/16，可分配 65,534 个 Fake-IP
```

### Fake-IP 映射表

mihomo 内部维护一个 Fake-IP ↔ 真实域名的映射表：

```
┌────────────────────────────────────┐
│        Fake-IP 映射表               │
├────────────────────────────────────┤
│  198.18.0.1    → example.com       │
│  198.18.0.2    → google.com        │
│  198.18.0.3    → github.com        │
│  198.18.0.4    → youtube.com       │
│  ...                               │
└────────────────────────────────────┘
```

当应用尝试连接 `198.18.0.1:443` 时，mihomo 查找映射表得到 `example.com`，然后基于域名执行路由规则。

### Fake-IP 与正常 DNS 模式对比

| 特性 | Fake-IP 模式 | 正常 DNS 模式（Redir-Host） |
|------|-------------|---------------------------|
| DNS 响应速度 | 即时 | 需要等待上游 DNS |
| 规则匹配依据 | 域名 | IP 地址 |
| 规则精确度 | 高（域名级） | 低（IP 级，CDN 影响大） |
| DNS 泄露风险 | 无（不依赖外部 DNS） | 有（需要上游 DNS） |
| 兼容性 | 大多数场景 | 所有场景 |
| 不适用场景 | 需要真实 IP 的应用 | 无 |

### Fake-IP 过滤

某些域名不应使用 Fake-IP（如局域网域名、特定服务域名）：

```yaml
dns:
  fake-ip-filter:
    - "*.lan"                     # 局域网域名
    - "*.local"                   # mDNS 域名
    - "*.home.arpa"               # 家庭网络域名
    - "localhost.ptlogin2.qq.com" # 特定服务
```

这些域名将使用正常 DNS 解析，返回真实 IP 地址。

## DNS 劫持验证与排障

### 验证 DNS 劫持是否生效

```bash
# 使用 nslookup 测试
nslookup example.com

# 预期输出（Fake-IP 模式）:
# Server:  198.18.0.1
# Address: example.com → 198.18.x.x

# 预期输出（正常模式）:
# Server:  mihomo DNS
# Address: example.com → 93.184.216.34 (真实 IP)
```

### 常见排障步骤

1. **检查 DNS 劫持规则**: 确认 `dns-hijack` 配置正确
2. **检查 DNS 模块**: 确认 `dns.enable: true`
3. **检查 Fake-IP 范围**: 确认 `fake-ip-range` 未与其他网络冲突
4. **查看 DNS 日志**: mihomo 日志会显示 DNS 查询处理情况
5. **抓包验证**: 使用 `tcpdump -i tun0 port 53` 查看 DNS 流量