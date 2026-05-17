---
title: "mihomo 规则集"
category: "mihomo"
type: ref
layer: ref
module: "rules/provider"
source: "mihomo-Meta"
tags: [mihomo, rules, rule-set, 规则集]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo 规则集

规则集用于批量管理大量规则，支持远程更新。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/provider/rule_set.go` | RULE-SET 规则实现 |
| `rules/provider/provider.go` | 规则提供者 |
| `rules/provider/classical_strategy.go` | classical 格式处理 |
| `rules/provider/domain_strategy.go` | domain 格式处理 |
| `rules/provider/ipcidr_strategy.go` | ipcidr 格式处理 |
| `rules/provider/mrs_reader.go` | MRS 格式读取 |

## RuleSet 结构

```go
type RuleSet struct {
    common.Base
    ruleProviderName string
    adapter          string
    isSrc            bool
    noResolveIP      bool
}

func (rs *RuleSet) Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string) {
    if provider, ok := rs.getProvider(); ok {
        if rs.isSrc {
            metadata.SwapSrcDst()
            defer metadata.SwapSrcDst()
            helper.ResolveIP = nil
        } else if rs.noResolveIP {
            helper.ResolveIP = nil
        }
        return provider.Match(metadata, helper), rs.adapter
    }
    return false, ""
}

func NewRuleSet(ruleProviderName string, adapter string, isSrc bool, noResolveIP bool) (*RuleSet, error) {
    rs := &RuleSet{
        ruleProviderName: ruleProviderName,
        adapter:          adapter,
        isSrc:            isSrc,
        noResolveIP:      noResolveIP,
    }
    return rs, nil
}
```

## YAML 配置

### rule-providers 配置

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt"
    path: ./ruleset/reject.yaml
    interval: 86400
    
  proxy:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400
    
  direct:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/direct.txt"
    path: ./ruleset/direct.yaml
    interval: 86400
    
  cncidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cncidr.txt"
    path: ./ruleset/cncidr.yaml
    interval: 86400
```

### RULE-SET 使用

```yaml
rules:
  - RULE-SET,reject,REJECT
  - RULE-SET,proxy,Proxy
  - RULE-SET,direct,DIRECT
  - RULE-SET,cncidr,DIRECT,no-resolve
```

## provider 类型

| 类型 | 说明 |
|------|------|
| `http` | 从远程 URL 加载 |
| `file` | 从本地文件加载 |

## behavior 类型

| behavior | 说明 | 规则格式 |
|----------|------|----------|
| `domain` | 域名规则集 | `+.google.com` |
| `ipcidr` | IP CIDR 规则集 | `91.108.4.0/22` |
| `classical` | 经典规则集 | 完整规则格式 |

## domain behavior 格式

```yaml
payload:
  - "+.google.com"
  - "+.youtube.com"
  - "+.github.com"
```

## ipcidr behavior 格式

```yaml
payload:
  - "91.108.4.0/22,no-resolve"
  - "91.108.8.0/22,no-resolve"
  - "149.154.160.0/20,no-resolve"
```

## classical behavior 格式

```yaml
payload:
  - DOMAIN,www.google.com,Proxy
  - DOMAIN-SUFFIX,google.com,Proxy
  - IP-CIDR,91.108.4.0/22,Proxy,no-resolve
```

## 配置字段说明

### rule-providers 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | 类型：http/file |
| `behavior` | string | 格式：domain/ipcidr/classical |
| `url` | string | 远程 URL（http 类型） |
| `path` | string | 本地路径 |
| `interval` | int | 更新间隔（秒） |
| `format` | string | 格式：yaml/text/mrs |

### RULE-SET 字段

| 格式 | 说明 |
|------|------|
| `RULE-SET,规则集名,策略` | 基本格式 |
| `RULE-SET,规则集名,策略,no-resolve` | 不解析 DNS |

## 完整配置示例

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt"
    path: ./ruleset/reject.yaml
    interval: 86400
    
  proxy:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400
    
  direct:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/direct.txt"
    path: ./ruleset/direct.yaml
    interval: 86400
    
  cncidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cncidr.txt"
    path: ./ruleset/cncidr.yaml
    interval: 86400

rules:
  - RULE-SET,reject,REJECT
  - RULE-SET,proxy,Proxy
  - RULE-SET,direct,DIRECT
  - RULE-SET,cncidr,DIRECT,no-resolve
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```

## 相关文档

- [[overview]] - 规则系统概述
- [[geosite]] - GeoSite 规则
- [[geoip]] - GeoIP 规则
- [[logic]] - 逻辑规则