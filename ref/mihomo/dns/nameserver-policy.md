---
title: "mihomo DNS nameserver-policy 配置"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, nameserver-policy, DNS策略]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS nameserver-policy 配置

`nameserver-policy` 定义特定域名使用的 DNS 服务器。

## 源码位置

| 文件 | 描述 |
|------|------|
| `dns/policy.go` | DNS 策略实现 |
| `dns/resolver.go` | Resolver 策略处理 |

## dnsPolicy 接口

```go
type dnsPolicy interface {
    Match(domain string) []dnsClient
}

type domainMatcherPolicy struct {
    matcher    C.DomainMatcher
    dnsClients []dnsClient
}

func (p domainMatcherPolicy) Match(domain string) []dnsClient {
    if p.matcher.MatchDomain(domain) {
        return p.dnsClients
    }
    return nil
}
```

## YAML 配置

### 基本配置

```yaml
dns:
  nameserver-policy:
    "*.google.com": https://dns.google/dns-query
    "*.github.com": https://dns.google/dns-query
```

### 使用域名匹配器

```yaml
dns:
  nameserver-policy:
    "geosite:google":
      - https://dns.google/dns-query
    "geosite:cn":
      - https://dns.alidns.com/dns-query
      - https://doh.pub/dns-query
```

### 广告域名处理

```yaml
dns:
  nameserver-policy:
    "geosite:category-ads-all":
      - rcode: success
      - rcode: name_error
```

## 配置格式

| 格式 | 说明 |
|------|------|
| `"域名": DNS服务器` | 单域名单 DNS |
| `"域名": [DNS列表]` | 单域名多 DNS |
| `"geosite:分类": [DNS列表]` | GeoSite 匹配 |

## rcode 处理

```yaml
dns:
  nameserver-policy:
    "geosite:category-ads-all":
      - rcode: success      # 返回成功（空结果）
      - rcode: name_error   # 返回域名不存在
      - rcode: refused      # 返回拒绝
```

| rcode | 说明 |
|-------|------|
| `success` | 返回成功响应 |
| `name_error` | NXDOMAIN |
| `refused` | REFUSED |

## 完整配置示例

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  
  # 默认 DNS
  nameserver:
    - https://dns.alidns.com/dns-query
    
  # DNS 策略
  nameserver-policy:
    # Google 使用海外 DNS
    "geosite:google":
      - https://dns.google/dns-query
      
    # GitHub 使用海外 DNS
    "geosite:github":
      - https://dns.google/dns-query
      
    # Telegram 使用海外 DNS
    "geosite:telegram":
      - https://dns.google/dns-query
      
    # 国内域名使用国内 DNS
    "geosite:cn":
      - https://dns.alidns.com/dns-query
      - https://doh.pub/dns-query
      
    # 广告域名拒绝
    "geosite:category-ads-all":
      - rcode: name_error
      
    # 特定域名
    "*.openai.com": https://dns.google/dns-query
```

## 匹配优先级

```
1. nameserver-policy 匹配
2. fake-ip-filter 检查
3. nameserver 查询
4. fallback-filter 判断
5. fallback 查询
```

## 相关文档

- [[overview]] - DNS 配置概述
- [[ref/mihomo/dns/servers|servers]] - DNS 服务器配置
- [[ref/mihomo/dns/fallback|fallback]] - fallback DNS