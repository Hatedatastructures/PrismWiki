---
title: "mihomo DNS fallback 配置"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, fallback, 备用DNS]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS fallback 配置

`fallback` 定义备用 DNS 服务器，处理 nameserver 返回异常结果的情况。

## 源码位置

| 文件 | 描述 |
|------|------|
| `dns/resolver.go` | Resolver fallback 处理 |

## Resolver fallback 结构

```go
type Resolver struct {
    main                  []dnsClient       // nameserver
    fallback              []dnsClient       // fallback DNS
    fallbackDomainFilters []C.DomainMatcher // 域名过滤器
    fallbackIPFilters     []C.IpMatcher     // IP 过滤器
}
```

## ipExchange 方法

```go
func (r *Resolver) ipExchange(ctx context.Context, m *D.Msg) (msg *D.Msg, err error) {
    // 先用 nameserver 查询
    msgCh := r.asyncExchange(ctx, r.main, m)
    
    if r.fallback == nil || len(r.fallback) == 0 {
        res := <-msgCh
        return res.Msg, res.Error
    }
    
    res := <-msgCh
    if res.Error == nil {
        if ips := msgToIP(res.Msg); len(ips) != 0 {
            // 检查 IP 是否命中 fallback-filter
            shouldNotFallback := lo.EveryBy(ips, func(ip netip.Addr) bool {
                return !r.shouldIPFallback(ip)
            })
            if shouldNotFallback {
                return res.Msg, res.Error
            }
        }
    }
    
    // 使用 fallback 重新查询
    res = <-r.asyncExchange(ctx, r.fallback, m)
    return res.Msg, res.Error
}
```

## YAML 配置

```yaml
dns:
  nameserver:
    - https://dns.alidns.com/dns-query
    
  fallback:
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
    - https://dns.nextdns.io/dns-query
    
  fallback-filter:
    geoip: true
    geoip-code: CN
```

## fallback 工作流程

```
DNS 查询 google.com
    │
    ▼
nameserver 查询（国内 DNS）
结果: 污染的 IP 或非 CN IP
    │
    ▼
fallback-filter 判断
命中 geoip/IPCIDR/domain 过滤器
    │
    ▼
fallback 查询（海外 DNS）
结果: 正确的 IP
```

## 配置说明

| 字段 | 说明 |
|------|------|
| `fallback` | 备用 DNS 服务器列表 |
| `fallback-filter` | 决定何时使用 fallback |

## 常用 fallback DNS

```yaml
fallback:
  # Google DNS
  - https://dns.google/dns-query
  - tls://dns.google:853
  
  # Cloudflare DNS
  - https://cloudflare-dns.com/dns-query
  - tls://1.1.1.1:853
  
  # NextDNS
  - https://dns.nextdns.io/dns-query
  - quic://dns.nextdns.io:853
```

## fallback-filter

参见 [[fallback-filter]]：

```yaml
fallback-filter:
  geoip: true
  geoip-code: CN
  ipcidr:
    - 240.0.0.0/4
    - 0.0.0.0/32
  domain:
    - "+.google.com"
```

## 完整配置示例

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  
  # 国内 DNS（直连）
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
    
  # 海外 DNS（代理）
  fallback:
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
    
  # fallback 过滤条件
  fallback-filter:
    geoip: true
    geoip-code: CN
    ipcidr:
      - 240.0.0.0/4
      - 0.0.0.0/32
    domain:
      - "+.google.com"
      - "+.github.com"
```

## 相关文档

- [[overview]] - DNS 配置概述
- [[servers]] - DNS 服务器配置
- [[fallback-filter]] - fallback 过滤