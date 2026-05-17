---
title: "mihomo DNS 配置概述"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, DNS配置, 解析]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS 配置概述

mihomo DNS 模块处理域名解析，防止 DNS 泄露并优化解析速度。

## 源码位置

| 文件 | 描述 |
|------|------|
| `dns/resolver.go` | DNS 解析器 |
| `dns/policy.go` | DNS 策略 |
| `dns/server.go` | DNS 服务器 |
| `dns/enhancer.go` | fake-ip 增强 |
| `dns/doh.go` | DoH 客户端 |
| `dns/dot.go` | DoT 客户端 |
| `dns/doq.go` | DoQ 客户端 |
| `dns/client.go` | DNS 客户端接口 |

## DNS 模块类型

| 类型 | 说明 | 源码 |
|------|------|------|
| [[servers]] | DNS 服务器配置 | `resolver.go` |
| [[enable]] | 启用 DNS | 配置项 |
| [[listen]] | 监听地址 | `server.go` |
| [[enhanced-mode]] | 增强模式 | `enhancer.go` |
| [[fake-ip-filter]] | fake-ip 过滤 | `enhancer.go` |
| [[nameserver-policy]] | DNS 策略 | `policy.go` |
| [[fallback]] | fallback DNS | `resolver.go` |
| [[fallback-filter]] | fallback 过滤 | `resolver.go` |

## Resolver 结构

```go
type Resolver struct {
    ipv6                  bool
    ipv6Timeout           time.Duration
    main                  []dnsClient
    fallback              []dnsClient
    fallbackDomainFilters []C.DomainMatcher
    fallbackIPFilters     []C.IpMatcher
    group                 singleflight.Group[*D.Msg]
    cache                 dnsCache
    policy                []dnsPolicy
    defaultResolver       *Resolver
}
```

## YAML 配置结构

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
  ipv6: false
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - "*.lan"
  nameserver:
    - https://dns.alidns.com/dns-query
  fallback:
    - https://dns.google/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN
  nameserver-policy:
    "*.google.com": https://dns.google/dns-query
```

## DNS 模式说明

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `fake-ip` | 返回虚假 IP | 推荐，性能好 |
| `redir-host` | 真实解析 | 需要真实 IP |

## 客户端类型

```go
type dnsClient interface {
    ExchangeContext(ctx context.Context, m *D.Msg) (msg *D.Msg, err error)
    Address() string
    ResetConnection()
}
```

支持：
- UDP DNS
- TCP DNS
- DoH (`dns/doh.go`)
- DoT (`dns/dot.go`)
- DoQ (`dns/doq.go`)

## NameServer 结构

```go
type NameServer struct {
    Net          string
    Addr         string
    ProxyAdapter C.ProxyAdapter
    ProxyName    string
    Params       map[string]string
    PreferH3     bool
}
```

## 相关文档

- [[servers]] - DNS 服务器配置
- [[enable]] - 启用 DNS
- [[listen]] - 监听地址
- [[enhanced-mode]] - 增强模式
- [[fake-ip-filter]] - fake-ip 过滤
- [[nameserver-policy]] - DNS 策略
- [[fallback]] - fallback DNS
- [[fallback-filter]] - fallback 过滤