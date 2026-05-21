---
title: DNS
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, dns]
---
# DNS 代理协议

DNS 代理是 mihomo 中用于处理 DNS 查询请求的特殊协议类型。它不转发实际的网络连接，而是将 DNS 查询请求重定向到 mihomo 的 DNS 解析引擎，由引擎根据配置的策略进行解析并返回结果。该协议是实现 DNS 劫持、Fake-IP 模式和 DNS 分流的核心组件。

## 技术概述

DNS 代理协议的核心功能是将系统或应用的 DNS 查询请求拦截并交给 mihomo 内置的 DNS 服务器处理。其特性包括：

- **DNS 查询劫持**：拦截系统 DNS 请求，由 mihomo 统一处理
- **多协议支持**：支持 UDP DNS、TCP DNS、DNS-over-HTTPS（DoH）、DNS-over-TLS（DoT）
- **Fake-IP 模式**：返回占位 IP 地址，按需解析真实地址
- **DNS 分流**：不同域名使用不同上游 DNS 服务器
- **域名嗅探**：从流量中嗅探真实域名并用于 DNS 解析
- **缓存机制**：DNS 响应缓存，减少重复查询延迟

DNS 代理在 mihomo 中作为 outbound 协议存在，与 Direct 和 Reject 一起构成规则引擎的三大内置协议。

## DNS 代理原理

### 工作流程

DNS 代理的处理流程如下：

```
应用发起 DNS 查询（UDP/TCP :53）
    ↓
TUN / 透明代理捕获
    ↓
mihomo 规则引擎匹配 → DNS 规则
    ↓
DNS 代理接收查询请求
    ↓
查询 mihomo DNS 解析引擎
    ↓
DNS 引擎根据 nameserver-policy 选择上游
    ↓
向上游 DNS 发起查询（DoH / DoT / UDP）
    ↓
获取 DNS 响应
    ↓
返回给应用
```

与普通代理不同，DNS 代理不需要建立到目标服务器的连接。它的职责是将 DNS 查询交给 mihomo 的 DNS 引擎，由引擎完成解析后返回结果。

### DNS 查询劫持

在 TUN 模式下，DNS 查询劫持的机制如下：

1. 系统配置 TUN 设备为默认路由，所有流量（包括 DNS 查询）进入 TUN
2. mihomo 监听 TUN 设备上的 UDP 53 端口和 TCP 53 端口
3. 当 DNS 查询到达时，DNS 代理协议将其重定向到 DNS 引擎
4. DNS 引擎使用配置的上游服务器进行解析

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053        # DNS 监听地址
  enhanced-mode: fake-ip       # 增强模式：fake-ip
  fake-ip-range: 198.18.0.1/16 # Fake-IP 地址池

  nameserver:
    - https://1.1.1.1/dns-query    # Cloudflare DoH
    - https://8.8.8.8/dns-query    # Google DoH

  fallback:
    - https://dns.google/dns-query # 备用 DNS
```

### TCP DNS 与 UDP DNS

| 特性 | UDP DNS | TCP DNS |
|------|---------|---------|
| 端口 | 53 | 53 |
| 最大响应大小 | 512 字节（传统限制） | 无限制 |
| 可靠性 | 不可靠（可能丢包） | 可靠 |
| EDNS0 | 支持扩展（>512 字节） | 原生支持大响应 |
| 使用场景 | 大多数普通查询 | DNSSEC、大记录集 |

mihomo 的 DNS 代理同时支持两种传输方式，并根据查询类型自动选择。

## DNS-over-X 封装流程

mihomo 支持多种加密 DNS 传输协议，统称为 DNS-over-X：

### DNS-over-HTTPS (DoH)

DoH 将 DNS 查询封装在 HTTPS 请求中：

```
DNS Query
    ↓
序列化为 DNS Wire Format
    ↓
Base64URL 编码
    ↓
HTTPS GET /dns-query?dns=<base64url-data>
    ↓
TLS 加密传输
    ↓
DoH 服务器
    ↓
解析 DNS 查询
    ↓
返回 DNS Wire Format 响应
    ↓
HTTPS Response
    ↓
解码 Base64URL
    ↓
反序列化 DNS 响应
    ↓
返回给应用
```

DoH 的优势在于 DNS 流量与正常 HTTPS 流量无法区分，提供了额外的隐私保护。

### DNS-over-TLS (DoT)

DoT 在 TLS 之上直接传输 DNS 查询：

```
DNS Query
    ↓
序列化为 DNS Wire Format
    ↓
TLS 握手
    ↓
加密传输 DNS Wire Format
    ↓
DoT 服务器（端口 853）
    ↓
解密并解析
    ↓
返回加密 DNS 响应
    ↓
解密并返回给应用
```

DoT 的端口固定为 853，相比 DoH 更容易被识别和封锁，但在性能上略优于 DoH。

### DNS-over-QUIC (DoQ)

DoQ 使用 QUIC 协议传输 DNS 查询：

```
DNS Query
    ↓
序列化为 DNS Wire Format
    ↓
QUIC 连接建立（0-RTT 可选）
    ↓
加密传输
    ↓
DoQ 服务器（端口 853 或 443）
    ↓
返回响应
```

DoQ 结合了 QUIC 的低延迟和加密优势，是最新的 DNS 加密标准。

### 上游 DNS 配置示例

```yaml
dns:
  nameserver:
    # 标准 UDP DNS
    - udp://223.5.5.5:53
    - udp://119.29.29.29:53

    # DNS-over-TLS
    - tls://dns.alidns.com:853
    - tls://1.1.1.1:853

    # DNS-over-HTTPS
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
    - https://1.1.1.1/dns-query

    # HTTP3 DoH
    - h3://dns.google/dns-query
```

## Fake-IP 模式

### 原理

Fake-IP 模式是 mihomo 独有的 DNS 增强功能。其核心思想是：

1. 当应用发起 DNS 查询时，mihomo 立即返回一个 Fake-IP（如 `198.18.x.x` 范围内的地址）
2. 应用使用该 Fake-IP 发起连接
3. mihomo 拦截连接，通过 Fake-IP 映射表找到真实域名
4. 根据域名匹配代理规则，选择对应的代理或直连
5. 在代理链末端或直连时再执行真实的 DNS 解析

```
应用 DNS 查询 example.com
    ↓
mihomo 立即返回 198.18.0.1（Fake-IP）
    ↓
应用连接 198.18.0.1:443
    ↓
mihomo 拦截：198.18.0.1 → example.com
    ↓
规则匹配：example.com → PROXY
    ↓
代理节点执行真实 DNS 解析
    ↓
连接真实服务器
```

### 优势

| 特性 | 传统模式 | Fake-IP 模式 |
|------|----------|-------------|
| DNS 延迟 | 需要等待真实解析 | 立即返回 |
| DNS 污染 | 本地 DNS 可能被污染 | 由代理节点或可信 DNS 解析 |
| 规则匹配 | 需要域名嗅探 | 直接通过 Fake-IP 映射 |
| 首次连接速度 | DNS 延迟 + 连接延迟 | 仅连接延迟 |

### 配置

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16    # 65534 个可用 Fake-IP
  fake-ip-filter:                # 不走 Fake-IP 的域名
    - "*.lan"
    - "*.local"
    - localhost.ptlogin2.qq.com
```

### Fake-IP 过滤器

Fake-IP 过滤器用于排除不需要 Fake-IP 的域名：

```yaml
dns:
  fake-ip-filter:
    # 本地域名
    - "*.lan"
    - "*.localdomain"

    # 路由器
    - "router.asus.com"

    # 特定服务
    - "localhost.ptlogin2.qq.com"

    # 所有 mDNS 域名
    - "*.local"
    - "*.home.arpa"
```

被过滤器匹配的域名会走正常 DNS 解析流程，返回真实 IP 地址。

## DNS 分流

mihomo 支持针对不同域名使用不同的上游 DNS 服务器：

```yaml
dns:
  nameserver-policy:
    # 国内域名使用国内 DNS
    "geosite:cn":
      - https://dns.alidns.com/dns-query
      - https://doh.pub/dns-query

    # Google 服务使用 Google DNS
    "geosite:google":
      - https://8.8.8.8/dns-query

    # 特定域名
    "example.com":
      - https://1.1.1.1/dns-query

    # 全局默认
    "*":
      - https://1.1.1.1/dns-query
      - https://8.8.8.8/dns-query
```

DNS 分流确保：
- 国内域名通过国内 DNS 解析，获取最优 CDN 节点
- 海外域名通过海外 DNS 解析，避免 DNS 污染
- 特殊域名使用指定的 DNS 服务器

## 域名嗅探

mihomo 支持从流量中嗅探真实域名，用于 DNS 解析：

```yaml
sniffer:
  enable: true
  parse-pure-ip: true
  override-destination: true

  sniff:
    HTTP:
      ports: [80, 8080-8880]
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]

  force-domain:
    - "+.v2ex.com"

  skip-domain:
    - "Mijia Cloud"
    - "+.apple.com"
```

当连接的目标是纯 IP 地址时，mihomo 可以从 TLS SNI、HTTP Host 等字段中嗅探出真实域名，然后使用该域名进行 DNS 解析和规则匹配。

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/dns.go` | DNS 适配器，将查询转发到 DNS 引擎 |
| `dns/scheduler.go` | DNS 调度器，管理并发查询 |
| `dns/resolver.go` | DNS 解析器，执行真实查询 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | DNS 代理实现 |
| Resolver | 完全兼容 | DNS 解析集成 |
| Rules | 完全兼容 | DNS 规则匹配 |
| TUN | 完全兼容 | TUN 模式 DNS 劫持 |
| Sniffer | 完全兼容 | 域名嗅探集成 |

## YAML 配置示例

### 基本 DNS 代理

```yaml
proxies:
  - name: "DNS"
    type: dns
```

### 完整 DNS 配置

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  ipv6: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - "*.lan"
    - "*.local"

  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query

  fallback:
    - https://1.1.1.1/dns-query
    - tls://8.8.8.8:853

  fallback-filter:
    geoip: true
    geoip-code: CN
    geosite:
      - gfw

  nameserver-policy:
    "geosite:cn":
      - https://dns.alidns.com/dns-query
    "*":
      - https://1.1.1.1/dns-query
```

## 相关文档

- [[direct]] - Direct 协议
- [[reject]] - Reject 协议
- [[ref/mihomo/dns|dns]] - DNS 配置
- [[core/resolve/dns|dns]] - DNS 解析
