---
title: "mihomo GeoSite 规则"
category: "mihomo"
type: ref
layer: ref
module: "rules/common"
source: "mihomo-Meta"
tags: [mihomo, rules, geosite, 域名分类]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo GeoSite 规则

GeoSite 规则使用预编译的域名分类数据库进行匹配。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/common/geosite.go` | GeoSite 规则实现 |
| `component/geodata/` | GeoData 加载模块 |

## GEOSITE 结构

```go
type GEOSITE struct {
    Base
    country    string
    adapter    string
    recodeSize int
}

func (gs *GEOSITE) Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string) {
    return gs.MatchDomain(metadata.RuleHost()), gs.adapter
}

func (gs *GEOSITE) MatchDomain(domain string) bool {
    if len(domain) == 0 {
        return false
    }
    matcher, err := gs.GetDomainMatcher()
    if err != nil {
        return false
    }
    return matcher.ApplyDomain(domain)
}

func NewGEOSITE(country string, adapter string) (*GEOSITE, error) {
    if err := geodata.InitGeoSite(); err != nil {
        return nil, err
    }
    geoSite := &GEOSITE{
        country: country,
        adapter: adapter,
    }
    matcher, err := geoSite.GetDomainMatcher()
    if err != nil {
        return nil, err
    }
    log.Infoln("Finished initial GeoSite rule %s => %s, records: %d", country, adapter, matcher.Count())
    return geoSite, nil
}
```

## YAML 配置

### 基本配置

```yaml
rules:
  - GEOSITE,google,Proxy
  - GEOSITE,github,Proxy
  - GEOSITE,telegram,Proxy
  - GEOSITE,cn,DIRECT
```

### 广告拦截

```yaml
rules:
  - GEOSITE,category-ads-all,REJECT
```

## 常用 GeoSite 分类

| 分类 | 说明 |
|------|------|
| `cn` | 中国大陆网站域名 |
| `google` | Google 相关域名 |
| `github` | GitHub 相关域名 |
| `telegram` | Telegram 相关域名 |
| `microsoft` | Microsoft 相关域名 |
| `apple` | Apple 相关域名 |
| `facebook` | Facebook 相关域名 |
| `twitter` | Twitter 相关域名 |
| `amazon` | Amazon 相关域名 |
| `netflix` | Netflix 相关域名 |
| `disney` | Disney 相关域名 |
| `spotify` | Spotify 相关域名 |
| `category-ads-all` | 广告域名 |
| `geolocation-!cn` | 非中国域名 |

## 配置字段说明

| 规则类型 | 格式 | 说明 |
|----------|------|------|
| `GEOSITE` | `GEOSITE,分类名,策略` | GeoSite 域名分类匹配 |

## GeoData 配置

```yaml
geodata-mode: true
geox-url:
  geoip: "https://cdn.jsdelivr.net/gh/Loyalsoldier/geoip@release/geoip.dat"
  geosite: "https://cdn.jsdelivr.net/gh/Loyalsoldier/v2ray-rules-dat@release/geosite.dat"
  mmdb: "https://cdn.jsdelivr.net/gh/Loyalsoldier/geoip@release/Country.mmdb"
```

## 完整配置示例

```yaml
rules:
  # 广告拦截
  - GEOSITE,category-ads-all,REJECT
  
  # 代理域名
  - GEOSITE,google,Proxy
  - GEOSITE,github,Proxy
  - GEOSITE,telegram,Proxy
  - GEOSITE,openai,Proxy
  
  # 直连域名
  - GEOSITE,cn,DIRECT
  
  # 兜底
  - MATCH,Proxy
```

## DomainMatcher 接口

```go
type DomainMatcher interface {
    ApplyDomain(domain string) bool
    Count() int
}
```

## 相关文档

- [[overview]] - 规则系统概述
- [[geoip]] - GeoIP 规则
- [[domain]] - 域名规则
- [[rule-set]] - 规则集