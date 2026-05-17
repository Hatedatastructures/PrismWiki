---
title: "mihomo 端口规则"
category: "mihomo"
type: ref
layer: ref
module: "rules/common"
source: "mihomo-Meta"
tags: [mihomo, rules, port, 端口规则]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo 端口规则

端口规则根据请求端口进行匹配。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/common/port.go` | 端口规则实现 |

## Port 结构

```go
type Port struct {
    Base
    adapter    string
    port       string
    ruleType   C.RuleType
    portRanges utils.IntRanges[uint16]
}

func (p *Port) Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string) {
    targetPort := metadata.DstPort
    switch p.ruleType {
    case C.InPort:
        targetPort = metadata.InPort
    case C.SrcPort:
        targetPort = metadata.SrcPort
    }
    return p.portRanges.Check(targetPort), p.adapter
}

func NewPort(port string, adapter string, ruleType C.RuleType) (*Port, error) {
    portRanges, err := utils.NewUnsignedRanges[uint16](port)
    if err != nil {
        return nil, fmt.Errorf("%w, %w", errPayload, err)
    }
    return &Port{
        adapter:    adapter,
        port:       port,
        ruleType:   ruleType,
        portRanges: portRanges,
    }, nil
}
```

## YAML 配置

### DST-PORT（目标端口）

```yaml
rules:
  - DST-PORT,443,Proxy
  - DST-PORT,80,Proxy
  - DST-PORT,22,DIRECT
```

### SRC-PORT（源端口）

```yaml
rules:
  - SRC-PORT,12345,Proxy
```

### IN-PORT（入站端口）

```yaml
rules:
  - IN-PORT,7890,Proxy
```

### 端口范围

```yaml
rules:
  - DST-PORT,8000-9000,Proxy    # 茍围匹配
  - DST-PORT,80/443/8080,Proxy  # 多端口匹配
```

## 配置字段说明

| 规则类型 | 格式 | 说明 |
|----------|------|------|
| `DST-PORT` | `DST-PORT,端口,策略` | 目标端口匹配 |
| `SRC-PORT` | `SRC-PORT,端口,策略` | 源端口匹配 |
| `IN-PORT` | `IN-PORT,端口,策略` | 入站端口匹配 |

## 端口范围语法

| 语法 | 说明 | 示例 |
|------|------|------|
| 单端口 | 单个端口 | `80` |
| 范围 | 端口范围 | `8000-9000` |
| 多端口 | 多个端口 | `80/443/8080` |

## 常用端口规则

```yaml
rules:
  # HTTP/HTTPS
  - DST-PORT,80,Proxy
  - DST-PORT,443,Proxy
  
  # SSH 直连
  - DST-PORT,22,DIRECT
  
  # 游戏端口
  - DST-PORT,27000-28000,DIRECT
  
  # 多端口
  - DST-PORT,80/443/8080,Proxy
```

## DSCP 规则

```yaml
rules:
  - DSCP,4,Proxy
```

## 相关文档

- [[overview]] - 规则系统概述
- [[process]] - 进程规则