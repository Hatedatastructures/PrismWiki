---
title: "mihomo DNS fallback-filter 配置"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, fallback-filter, 过滤条件]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS fallback-filter 配置

`fallback-filter` 定义何时使用 fallback DNS 的判断条件。

## 源码位置

| 文件 | 描述 |
|------|------|
| `dns/resolver.go` | Resolver 过滤处理 |
| `rules/common/geoip.go` | GeoIP 匹配 |

## 过滤判断逻辑

```go
func (r *Resolver) shouldIPFallback(ip netip.Addr) bool {
    for _, filter := range r.fallbackIPFilters {
        if filter.MatchIp(ip) {
            return true
        }
    }
    return false
}

func (r *Resolver) shouldOnlyQueryFallback(m *D.Msg) bool {
    domain := msgToDomain(m)
    for _, df := range r.fallbackDomainFilters {
        if df.MatchDomain(domain) {
            return true
        }
    }
    return false
}
```

## YAML 配置

```yaml
dns:
  fallback-filter:
    geoip: true
    geoip-code: CN
    geoip-url: "https://example.com/Country.mmdb"
    ipcidr:
      - 240.0.0.0/4
      - 0.0.0.0/32
    domain:
      - "+.google.com"
      - "+.github.com"
```

## 过滤条件说明

| 条件 | 说明 | 判断时机 |
|------|------|----------|
| `geoip` | IP 地理位置判断 | nameserver 返回结果后 |
| `geoip-code` | 目标国家代码 | IP 不属于此国家时用 fallback |
| `ipcidr` | IP 黑名单段 | IP 在黑名单时用 fallback |
| `domain` | 域名黑名单 | 直接用 fallback，不经过 nameserver |

## geoip 配置

```yaml
fallback-filter:
  geoip: true
  geoip-code: CN
```

判断逻辑：
- nameserver 返回的 IP 不属于 CN 时，使用 fallback

## geoip-url 配置

```yaml
fallback-filter:
  geoip: true
  geoip-code: CN
  geoip-url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/geoip@release/Country.mmdb"
```

指定 GeoIP 数据库 URL。

## ipcidr 配置

```yaml
fallback-filter:
  ipcidr:
    - 240.0.0.0/4      # Class E Reserved
    - 0.0.0.0/32       # 空地址
    - 127.0.0.1/32     # 本地回环
    - 100.64.0.0/10    # Carrier-grade NAT
```

典型污染 IP 段：

| IP 段 | 含义 | 说明 |
|-------|------|------|
| `0.0.0.0/32` | 空地址 | DNS 污染常返回 |
| `127.0.0.1/32` | 本地回环 | 部分污染返回 |
| `240.0.0.0/4` | Reserved | 污染常返回此段 |

## domain 配置

```yaml
fallback-filter:
  domain:
    - "+.google.com"
    - "+.youtube.com"
    - "+.github.com"
    - "+.facebook.com"
    - "+.twitter.com"
    - "+.telegram.org"
```

域名在黑名单时，直接使用 fallback，不经过 nameserver。

## 完整配置示例

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  
  nameserver:
    - https://dns.alidns.com/dns-query
    
  fallback:
    - https://dns.google/dns-query
    
  fallback-filter:
    # GeoIP 判断
    geoip: true
    geoip-code: CN
    
    # IP 黑名单
    ipcidr:
      - 240.0.0.0/4
      - 0.0.0.0/32
      - 127.0.0.1/32
      - 100.64.0.0/10
      
    # 域名黑名单
    domain:
      - "+.google.com"
      - "+.youtube.com"
      - "+.github.com"
      - "+.facebook.com"
      - "+.twitter.com"
      - "+.telegram.org"
```

## 判断流程

```
DNS 查询到达
    │
    ▼
检查 domain 黑名单
    │
    ├─ 命名 ─> 直接用 fallback
    │
    └─ 未命中 ─> 用 nameserver
                  │
                  ▼
               检查返回的 IP
                  │
                  ├─ geoip 非 CN ─> 用 fallback
                  │
                  ├─ ipcidr 黑名单 ─> 用 fallback
                  │
                  └─ 正常 ─> 用 nameserver 结果
```

## 相关文档

- [[overview]] - DNS 配置概述
- [[fallback]] - fallback DNS
- [[servers]] - DNS 服务器配置