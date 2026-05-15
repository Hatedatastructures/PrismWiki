---
title: "mihomo DNS 处理机制"
created: 2026-05-13
updated: 2026-05-13
type: client
tags: [dns, mihomo, client, resolve, configuration]
related:
  - "[[client/mihomo-meta]]"
  - "[[client/mihomo-clash-config]]"
  - "[[client/mihomo-rules]]"
  - "[[resolve]]"
  - "[[dev/tcp]]"
  - "[[docs/protocol/proxy-protocols]]"
  - "[[agent]]"
---

# mihomo DNS 处理机制

mihomo（原 Clash.Meta）内置了完整的 DNS 解析模块，能够在代理层面拦截、处理和转发 DNS 查询，防止 DNS 泄露并提升解析速度。

## DNS 架构概览

mihomo 支持三种 DNS 处理模式（enhanced-mode）：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `fake-ip` | 返回虚假 IP，连接时还原域名 | 推荐，性能好，防泄露 |
| `redir-host` | 真实解析，缓存结果 | 需要真实 IP 的场景 |
| `fake-ip-filter` | 对特定域名使用真实解析 | 混合模式 |

## fake-ip 模式

fake-ip 是 mihomo 最核心的 DNS 机制：

1. 客户端向 mihomo DNS 发起域名查询
2. mihomo 从预分配的 fake-ip 池（默认 `198.18.0.0/15`）返回一个虚假 IP
3. 客户端用该虚假 IP 发起连接
4. mihomo 拦截连接，根据虚假 IP 还原原始域名
5. 通过代理或直连规则处理该域名

### 优势

- **零 DNS 泄露**：真实 DNS 查询始终通过代理发出
- **低延迟**：本地立即返回结果，无需等待真实解析
- **域名还原**：确保代理服务器收到正确的 SNI / Host

### 局限

- 某些应用依赖 DNS 结果做地理判断（如 CDN 调度），fake-ip 可能导致非最优节点
- 需要 fake-ip-filter 排除不适合虚假 IP 的域名

## fake-ip-filter

```yaml
dns:
  fake-ip-filter:
    - "*.lan"
    - "*.local"
    - "dns.msftncsi.com"
    - "www.msftncsi.com"
    - "+.msftconnecttest.com"
    - "*.ntp.org.cn"
    - "time.*.com"
    - "time.*.gov"
```

这些域名会返回真实解析结果而非虚假 IP。常见需要排除的类型：

- 局域网域名（`.lan`, `.local`）
- 网络连通性检测（Windows NCSI, captive portal）
- 时间同步服务
- 需要真实 IP 的 CDN 域名

## DNS 服务器配置

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
  fallback:
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN
    ipcidr:
      - 240.0.0.0/4
    domain:
      - "+.google.com"
      - "+.facebook.com"
```

### nameserver 与 fallback 的关系

- `nameserver`：国内域名优先使用的 DNS 服务器
- `fallback`：被污染域名使用的海外 DNS 服务器
- `fallback-filter`：判断何时使用 fallback 的条件

判断逻辑：先查 nameserver，若结果命中 fallback-filter 条件（geoip 非 CN、在黑名单 IP 段、或域名匹配），则用 fallback 重新查询。

## DNS 规则

mihomo 支持独立的 DNS 规则（`dns` → `rules`），可以在 DNS 层面决定使用哪个 DNS 服务器或使用哪种 enhanced-mode：

```yaml
dns:
  rules:
    - domain-suffix: google.com
      nameserver:
        - https://dns.google/dns-query
    - geosite: cn
      nameserver:
        - https://dns.alidns.com/dns-query
    - domain-keyword: github
      fake-ip-filter:
        - "+.github.com"
```

详见 [[client/mihomo-rules]] 中关于规则语法的说明。

## DNS over HTTPS/TLS 通过代理

mihomo 的一个强大特性是 DNS 查询可以通过代理发出：

```yaml
dns:
  nameserver:
    - https://dns.google/dns-query  # 如果走代理，此查询通过代理服务器发出
  proxy-server-nameserver:
    - https://dns.alidns.com/dns-query  # 代理节点本身的域名解析
```

这确保了：
- 国内域名通过国内 DNS 直连解析
- 被污染域名通过代理使用海外 DNS
- 代理节点域名本身通过国内 DNS 解析（避免循环依赖）

## 与 Prism 的对比

Prism 的 [[resolve]] 模块采用不同的设计理念：

| 特性 | mihomo | Prism |
|------|--------|-------|
| DNS 模式 | fake-ip / redir-host | 独立 resolve 模块 |
| 配置方式 | 内嵌在主配置中 | 模块化配置 |
| 代理集成 | 紧耦合 | 解耦设计 |
| fake-ip | 原生支持 | 不直接支持 |

两种实现各有优劣。mihomo 的 fake-ip 模式在防泄露方面更彻底，而 Prism 的模块化设计更灵活。

## 平台注意事项

- **Windows**：TUN 模式下自动劫持系统 DNS
- **macOS**：需要注意系统 DNS 缓存（`sudo killall -HUP mDNSResponder`）
- **Android**：通过 VPN 模式劫持 DNS
- **路由器**：作为网关 DNS 使用时，需配置 DHCP 下发 mihomo 的 DNS 地址

## 常见问题

### DNS 泄露检测

使用在线工具（如 dnsleaktest.com）检测。确保：
1. enhanced-mode 设为 fake-ip
2. DNS 查询通过代理发出
3. 系统没有硬编码的 DNS 服务器

### 特定域名解析失败

在 fake-ip-filter 中添加该域名，或在 dns rules 中为其指定专用 nameserver。

## 相关页面

- [[client/mihomo-meta]] — mihomo 元数据与版本信息
- [[client/mihomo-clash-config]] — Clash 配置文件详解
- [[client/mihomo-rules]] — 规则系统详解
- [[resolve]] — Prism 解析模块
- [[dev/tcp]] — TCP 协议基础
- [[docs/protocol/proxy-protocols]] — 代理协议概览
