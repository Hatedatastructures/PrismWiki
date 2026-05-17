---
title: "mihomo 代理组配置"
category: "mihomo"
type: ref
layer: ref
module: "adapter/outboundgroup"
source: "mihomo-Meta/parser.go"
tags: [mihomo, proxy-groups, 配置, 参数]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo 代理组配置

mihomo 代理组的完整配置参数说明。

## 源码位置

- `adapter/outboundgroup/parser.go`

## GroupCommonOption 结构

```go
type GroupCommonOption struct {
    Name                string   `group:"name"`
    Type                string   `group:"type"`
    Proxies             []string `group:"proxies,omitempty"`
    Use                 []string `group:"use,omitempty"`
    URL                 string   `group:"url,omitempty"`
    Interval            int      `group:"interval,omitempty"`
    TestTimeout         int      `group:"timeout,omitempty"`
    MaxFailedTimes      int      `group:"max-failed-times,omitempty"`
    Lazy                bool     `group:"lazy,omitempty"`
    DisableUDP          bool     `group:"disable-udp,omitempty"`
    Filter              string   `group:"filter,omitempty"`
    ExcludeFilter       string   `group:"exclude-filter,omitempty"`
    ExcludeType         string   `group:"exclude-type,omitempty"`
    ExpectedStatus      string   `group:"expected-status,omitempty"`
    IncludeAll          bool     `group:"include-all,omitempty"`
    IncludeAllProxies   bool     `group:"include-all-proxies,omitempty"`
    IncludeAllProviders bool     `group:"include-all-providers,omitempty"`
    Hidden              bool     `group:"hidden,omitempty"`
    Icon                string   `group:"icon,omitempty"`
}
```

## YAML 配置示例

### select 类型完整配置

```yaml
proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - "节点A"
      - "节点B"
      - DIRECT
    disable-udp: false
    hidden: false
    icon: "https://example.com/icon.png"
```

### url-test 类型完整配置

```yaml
proxy-groups:
  - name: "自动选择"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    lazy: true
    timeout: 5000
    max-failed-times: 5
    expected-status: "200"
    proxies:
      - "节点A"
      - "节点B"
    filter: ""
    exclude-filter: ""
    exclude-type: ""
```

### 使用 provider

```yaml
proxy-groups:
  - name: "Provider节点"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    use:
      - provider1
      - provider2
    filter: "香港|日本"
    exclude-filter: "过期"
    exclude-type: "ss"
    include-all: false
    include-all-proxies: true
    include-all-providers: false
```

### 使用 include-all

```yaml
proxy-groups:
  - name: "所有节点"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    include-all: true
    # 等同于:
    # include-all-proxies: true
    # include-all-providers: true
```

## 配置字段详解

### 通用字段

| 字段 | 类型 | 说明 | 适用类型 |
|------|------|------|----------|
| `name` | string | 代理组名称 | 所有类型 |
| `type` | string | 代理组类型 | 所有类型 |
| `proxies` | []string | 节点列表 | 所有类型 |
| `use` | []string | 引用的 provider | 所有类型 |
| `disable-udp` | bool | 禁用 UDP | 所有类型 |
| `hidden` | bool | 面板隐藏 | 所有类型 |
| `icon` | string | 面板图标 | 所有类型 |

### 测试相关字段

| 字段 | 类型 | 说明 | 适用类型 |
|------|------|------|----------|
| `url` | string | 测试 URL | url-test/fallback/load-balance |
| `interval` | int | 测试间隔（秒） | url-test/fallback/load-balance |
| `timeout` | int | 测试超时（毫秒） | url-test/fallback/load-balance |
| `lazy` | bool | 懒加载模式 | url-test/fallback/load-balance |
| `max-failed-times` | int | 最大失败次数 | url-test/fallback/load-balance |
| `expected-status` | string | 预期 HTTP 状态码 | url-test/fallback/load-balance |

### url-test 专用字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `tolerance` | int | 延迟容差（毫秒） |

### load-balance 专用字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `strategy` | string | 负载均衡策略 |

### 过滤字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `filter` | string | 正则过滤节点 |
| `exclude-filter` | string | 正则排除节点 |
| `exclude-type` | string | 排除协议类型 |
| `include-all` | bool | 包含所有节点和 provider |
| `include-all-proxies` | bool | 包含所有 proxies |
| `include-all-providers` | bool | 包含所有 providers |

## 默认值说明

| 字段 | 默认值 |
|------|--------|
| `interval` | 300 |
| `timeout` | 5000 |
| `tolerance` | 150 |
| `lazy` | true |
| `expected-status` | "*" |
| `strategy` | consistent-hashing |

## 配置验证规则

```go
// parser.go
if len(groupOption.Proxies) == 0 && len(groupOption.Use) == 0 {
    return nil, fmt.Errorf("%s: %w", groupName, errMissProxy)
}
```

- `proxies` 或 `use` 至少需要填一个
- `name` 和 `type` 必填

## 已弃用字段

以下字段在新版本中已移除：

| 字段 | 说明 | 替代方案 |
|------|------|----------|
| `routing-mark` | 路由标记 | 在节点上设置 |
| `interface-name` | 接口名称 | 在节点上设置 |
| `dialer-proxy` | 链式代理 | 在节点上设置 |

## 相关文档

- [[overview]] - 代理组概述
- [[selector]] - select 代理组
- [[url-test]] - url-test 代理组
- [[fallback]] - fallback 代理组
- [[load-balance]] - load-balance 代理组
- [[../provider/overview|Provider 概述]]