---
title: "mihomo 规则系统概述"
category: "mihomo"
type: ref
layer: ref
module: "rules"
source: "mihomo-Meta"
tags: [mihomo, rules, 规则, 分流]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo 规则系统概述

mihomo 规则系统是流量分流的核心机制，根据请求属性匹配规则并决定流量去向。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/parser.go` | 规则解析器 |
| `rules/common/base.go` | 规则基类 |
| `rules/common/domain.go` | 域名规则 |
| `rules/common/ipcidr.go` | IP CIDR 规则 |
| `rules/common/port.go` | 端口规则 |
| `rules/common/process.go` | 进程规则 |
| `rules/common/geoip.go` | GeoIP 规则 |
| `rules/common/geosite.go` | GeoSite 规则 |
| `rules/logic/logic.go` | 逻辑规则 |
| `rules/provider/rule_set.go` | 规则集 |
| `rules/provider/provider.go` | 规则提供者 |

## 规则类型分类

### 域名规则

| 规则 | 说明 | 源码 |
|------|------|------|
| [[domain]] | 精确域名匹配 | `common/domain.go` |
| `DOMAIN-SUFFIX` | 域名后缀匹配 | `common/domain.go` |
| `DOMAIN-KEYWORD` | 域名关键词匹配 | `common/domain_keyword.go` |
| `DOMAIN-REGEX` | 域名正则匹配 | `common/domain_regex.go` |
| `DOMAIN-WILDCARD` | 域名通配符匹配 | `common/domain_wildcard.go` |

### IP 规则

| 规则 | 说明 | 源码 |
|------|------|------|
| [[ipcidr]] | IP CIDR 匹配 | `common/ipcidr.go` |
| `IP-CIDR6` | IPv6 CIDR 匹配 | `common/ipcidr.go` |
| [[geoip]] | GeoIP 匹配 | `common/geoip.go` |
| `IP-ASN` | IP ASN 匹配 | `common/ipasn.go` |
| `IP-SUFFIX` | IP 后缀匹配 | `common/ipsuffix.go` |

### 端口规则

| 规则 | 说明 | 源码 |
|------|------|------|
| [[port]] | 端口匹配 | `common/port.go` |
| `SRC-PORT` | 源端口匹配 | `common/port.go` |
| `DST-PORT` | 目标端口匹配 | `common/port.go` |
| `IN-PORT` | 入站端口匹配 | `common/port.go` |

### 进程规则

| 规则 | 说明 | 源码 |
|------|------|------|
| [[process]] | 进程名匹配 | `common/process.go` |
| `PROCESS-PATH` | 进程路径匹配 | `common/process.go` |
| `PROCESS-NAME-REGEX` | 进程名正则 | `common/process.go` |
| `PROCESS-PATH-REGEX` | 进程路径正则 | `common/process.go` |

### 规则集

| 规则 | 说明 | 源码 |
|------|------|------|
| [[geosite]] | GeoSite 域名集 | `common/geosite.go` |
| [[rule-set]] | 自定义规则集 | `provider/rule_set.go` |

### 逻辑规则

| 规则 | 说明 | 源码 |
|------|------|------|
| [[logic]] | AND/OR/NOT | `logic/logic.go` |
| `SUB-RULE` | 子规则引用 | `logic/logic.go` |

## YAML 配置示例

### 基础配置

```yaml
rules:
  - DOMAIN,www.google.com,Proxy
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-KEYWORD,google,Proxy
  - GEOIP,CN,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
  - DST-PORT,443,Proxy
  - PROCESS-NAME,chrome.exe,Proxy
  - MATCH,Proxy
```

### 规则格式

```yaml
rules:
  # 格式: 规则类型,匹配条件,策略,附加参数
  - 规则类型,匹配条件,代理组/策略,参数
```

## 规则基类

```go
type Base struct{}

type Rule interface {
    RuleType() C.RuleType
    Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string)
    Adapter() string
    Payload() string
}
```

## ParseRule 函数

```go
func ParseRule(tp, payload, target string, params []string, subRules map[string][]C.Rule) (parsed C.Rule, parseErr error) {
    switch tp {
    case "DOMAIN":
        parsed = RC.NewDomain(payload, target)
    case "DOMAIN-SUFFIX":
        parsed = RC.NewDomainSuffix(payload, target)
    case "DOMAIN-KEYWORD":
        parsed = RC.NewDomainKeyword(payload, target)
    case "GEOIP":
        isSrc, noResolve := RC.ParseParams(params)
        parsed, parseErr = RC.NewGEOIP(payload, target, isSrc, noResolve)
    case "IP-CIDR", "IP-CIDR6":
        isSrc, noResolve := RC.ParseParams(params)
        parsed, parseErr = RC.NewIPCIDR(payload, target, 
            RC.WithIPCIDRSourceIP(isSrc), RC.WithIPCIDRNoResolve(noResolve))
    case "RULE-SET":
        isSrc, noResolve := RC.ParseParams(params)
        parsed, parseErr = RP.NewRuleSet(payload, target, isSrc, noResolve)
    case "MATCH":
        parsed = RC.NewMatch(target)
    // ... 其他规则类型
    }
}
```

## 规则参数说明

| 参数 | 说明 | 适用规则 |
|------|------|----------|
| `no-resolve` | 不解析 DNS | IP-CIDR/GEOIP/RULE-SET |
| `src` | 匹配源 IP | IP-CIDR/GEOIP |

## 相关文档

- [[domain]] - 域名规则详解
- [[ipcidr]] - IP CIDR 规则详解
- [[port]] - 端口规则详解
- [[process]] - 进程规则详解
- [[geosite]] - GeoSite 规则详解
- [[geoip]] - GeoIP 规则详解
- [[rule-set]] - 规则集详解
- [[logic]] - 逻辑规则详解