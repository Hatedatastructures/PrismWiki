---
title: "mihomo 域名规则"
category: "mihomo"
type: ref
layer: ref
module: "rules/common"
source: "mihomo-Meta"
tags: [mihomo, rules, domain, 域名规则]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo 域名规则

域名规则是最常用的规则类型，根据目标域名进行匹配。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/common/domain.go` | DOMAIN 规则 |
| `rules/common/domain_suffix.go` | DOMAIN-SUFFIX 规则 |
| `rules/common/domain_keyword.go` | DOMAIN-KEYWORD 规则 |
| `rules/common/domain_regex.go` | DOMAIN-REGEX 规则 |
| `rules/common/domain_wildcard.go` | DOMAIN-WILDCARD 规则 |

## DOMAIN（精确匹配）

### 源码定义

```go
type Domain struct {
    Base
    domain  string
    adapter string
}

func (d *Domain) Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string) {
    return metadata.RuleHost() == d.domain, d.adapter
}

func NewDomain(domain string, adapter string) *Domain {
    return &Domain{
        domain:  strings.ToLower(domain),
        adapter: adapter,
    }
}
```

### YAML 配置

```yaml
rules:
  - DOMAIN,www.google.com,Proxy
  - DOMAIN,mail.google.com,Proxy
```

### 匹配逻辑

- `www.google.com` -> 匹配
- `google.com` -> 不匹配
- `mail.google.com` -> 不匹配

## DOMAIN-SUFFIX（后缀匹配）

### YAML 配置

```yaml
rules:
  - DOMAIN-SUFFIX,google.com,Proxy
```

### 匹配逻辑

- `google.com` -> 匹配
- `www.google.com` -> 匹配
- `mail.google.com` -> 匹配
- `google.co.jp` -> 不匹配

## DOMAIN-KEYWORD（关键词匹配）

### YAML 配置

```yaml
rules:
  - DOMAIN-KEYWORD,google,Proxy
```

### 匹配逻辑

- `google.com` -> 匹配
- `www.google.com` -> 匹配
- `googleusercontent.com` -> 匹配
- `notgoogle.com` -> 匹配（可能误匹配）

## DOMAIN-REGEX（正则匹配）

### 源码定义

```go
func NewDomainRegex(pattern string, adapter string) (*DomainRegex, error) {
    r, err := regexp2.Compile(pattern, regexp2.IgnoreCase)
    if err != nil {
        return nil, err
    }
    return &DomainRegex{
        pattern: pattern,
        adapter: adapter,
        regexp:  r,
    }, nil
}
```

### YAML 配置

```yaml
rules:
  - DOMAIN-REGEX,^google\.com$,Proxy
  - DOMAIN-REGEX,\.google\.com$,Proxy
```

## DOMAIN-WILDCARD（通配符匹配）

### YAML 配置

```yaml
rules:
  - DOMAIN-WILDCARD,*.google.com,Proxy
  - DOMAIN-WILDCARD,+.*.google.com,Proxy
```

### 通配符语法

| 语法 | 说明 |
|------|------|
| `*` | 单层通配符，匹配一层子域名 |
| `+` | 多层通配符，匹配任意层级子域名 |

## 配置字段说明

| 规则类型 | 格式 | 说明 |
|----------|------|------|
| `DOMAIN` | `DOMAIN,域名,策略` | 精确匹配 |
| `DOMAIN-SUFFIX` | `DOMAIN-SUFFIX,后缀,策略` | 后缀匹配 |
| `DOMAIN-KEYWORD` | `DOMAIN-KEYWORD,关键词,策略` | 关键词匹配 |
| `DOMAIN-REGEX` | `DOMAIN-REGEX,正则,策略` | 正则匹配 |
| `DOMAIN-WILDCARD` | `DOMAIN-WILDCARD,通配符,策略` | 通配符匹配 |

## 完整配置示例

```yaml
rules:
  # 精确域名
  - DOMAIN,www.google.com,Proxy
  
  # 域名后缀
  - DOMAIN-SUFFIX,google.com,Proxy
  - DOMAIN-SUFFIX,github.com,Proxy
  - DOMAIN-SUFFIX,telegram.org,Proxy
  
  # 域名关键词
  - DOMAIN-KEYWORD,google,Proxy
  - DOMAIN-KEYWORD,facebook,Proxy
  
  # 域名正则
  - DOMAIN-REGEX,^.*\.google\.com$,Proxy
  
  # 域名通配符
  - DOMAIN-WILDCARD,*.*.google.com,Proxy
```

## 相关文档

- [[overview]] - 规则系统概述
- [[geosite]] - GeoSite 规则
- [[rule-set]] - 规则集