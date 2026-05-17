---
title: "mihomo 逻辑规则"
category: "mihomo"
type: ref
layer: ref
module: "rules/logic"
source: "mihomo-Meta"
tags: [mihomo, rules, logic, AND, OR, NOT]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo 逻辑规则

逻辑规则组合多个条件实现复杂分流逻辑。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/logic/logic.go` | 逻辑规则实现 |

## Logic 结构

```go
type Logic struct {
    common.Base
    payload  string
    adapter  string
    ruleType C.RuleType
    rules    []C.Rule
    subRules map[string][]C.Rule
    payloadOnce sync.Once
}

func (logic *Logic) Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string) {
    switch logic.ruleType {
    case C.SubRules:
        if m, _ := logic.rules[0].Match(metadata, helper); m {
            return matchSubRules(metadata, logic.adapter, logic.subRules, helper)
        }
        return false, ""
    case C.NOT:
        if m, _ := logic.rules[0].Match(metadata, helper); !m {
            return true, logic.adapter
        }
        return false, ""
    case C.OR:
        for _, rule := range logic.rules {
            if m, _ := rule.Match(metadata, helper); m {
                return true, logic.adapter
            }
        }
        return false, ""
    case C.AND:
        for _, rule := range logic.rules {
            if m, _ := rule.Match(metadata, helper); !m {
                return false, logic.adapter
            }
        }
        return true, logic.adapter
    default:
        return false, ""
    }
}
```

## YAML 配置

### AND（所有条件满足）

```yaml
rules:
  - AND,((DOMAIN-SUFFIX,google.com),(DST-PORT,443)),Proxy
```

匹配逻辑：
- 域名是 google.com **且** 端口是 443 -> Proxy

### OR（任一条件满足）

```yaml
rules:
  - OR,((DOMAIN-KEYWORD,google),(DOMAIN-KEYWORD,facebook)),Proxy
```

匹配逻辑：
- 域名包含 google **或** 包含 facebook -> Proxy

### NOT（条件取反）

```yaml
rules:
  - NOT,((DOMAIN-KEYWORD,google)),Direct
```

匹配逻辑：
- 域名不包含 google -> Direct

### SUB-RULE（子规则引用）

```yaml
sub-rules:
  google:
    - DOMAIN-SUFFIX,google.com
    - DOMAIN-SUFFIX,youtube.com
    
rules:
  - SUB-RULE,google,Proxy
```

## 逻辑规则嵌套

```yaml
rules:
  # Google 服务且非广告域名走代理
  - AND,((OR,((DOMAIN-SUFFIX,google.com),(DOMAIN-SUFFIX,youtube.com))),(NOT,(DOMAIN-KEYWORD,ads))),Proxy
  
  # 国内域名但特定端口走代理
  - AND,((GEOSITE,cn),(DST-PORT,443)),Proxy
```

## 配置格式

| 规则类型 | 格式 | 说明 |
|----------|------|------|
| `AND` | `AND,((规则1),(规则2)),策略` | 所有条件满足 |
| `OR` | `OR,((规则1),(规则2)),策略` | 任一条件满足 |
| `NOT` | `NOT,((规则)),策略` | 条件取反 |
| `SUB-RULE` | `SUB-RULE,子规则名,策略` | 子规则引用 |

## Payload 解析

```go
func (logic *Logic) parsePayload(payload string, parseRule common.ParseRuleFunc) error {
    if !strings.HasPrefix(payload, "(") || !strings.HasSuffix(payload, ")") {
        return fmt.Errorf("payload format error")
    }
    
    subAllRanges, err := logic.format(payload)
    if err != nil {
        return err
    }
    
    subRanges := logic.findSubRuleRange(payload, subAllRanges)
    for _, subRange := range subRanges {
        subPayload := payload[subRange.start+1 : subRange.end]
        rule, err := logic.payloadToRule(subPayload, parseRule)
        if err != nil {
            return err
        }
        logic.rules = append(logic.rules, rule)
    }
    return nil
}
```

## Payload 格式化

规则 payload 必须使用括号包裹：
- `((规则1),(规则2))` - 正确格式
- `规则1,规则2` - 错误格式

## 完整配置示例

```yaml
sub-rules:
  google:
    - DOMAIN-SUFFIX,google.com
    - DOMAIN-SUFFIX,youtube.com
    - DOMAIN-SUFFIX,googleusercontent.com
    
rules:
  # AND: Google 且 HTTPS
  - AND,((DOMAIN-SUFFIX,google.com),(DST-PORT,443)),Proxy
  
  # OR: Google 或 Facebook
  - OR,((DOMAIN-KEYWORD,google),(DOMAIN-KEYWORD,facebook)),Proxy
  
  # NOT: 非 Google
  - NOT,((DOMAIN-KEYWORD,google)),Direct
  
  # SUB-RULE: 子规则引用
  - SUB-RULE,google,Proxy
  
  # 嵌套逻辑
  - AND,((OR,((DOMAIN-SUFFIX,google.com),(DOMAIN-SUFFIX,github.com))),(NOT,(DOMAIN-KEYWORD,ads))),Proxy
  
  # MATCH 兜底
  - MATCH,Proxy
```

## 不支持嵌套的规则

以下规则不能在逻辑规则中使用：
- `MATCH` - 兜底规则
- `SUB-RULE` - 不能嵌套 SUB-RULE

## 相关文档

- [[overview]] - 规则系统概述
- [[rule-set]] - 规则集