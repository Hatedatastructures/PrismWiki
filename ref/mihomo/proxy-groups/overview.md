---
title: "mihomo 代理组概述"
category: "mihomo"
type: ref
layer: ref
module: "adapter/outboundgroup"
source: "mihomo-Meta"
tags: [mihomo, proxy-groups, 配置, 代理组]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo 代理组概述

mihomo 代理组是流量控制的核心组件，定义节点选择策略。

## 源码位置

| 文件 | 描述 |
|------|------|
| `adapter/outboundgroup/parser.go` | 代理组解析器 |
| `adapter/outboundgroup/groupbase.go` | 代理组基类 |
| `adapter/outboundgroup/selector.go` | select 类型实现 |
| `adapter/outboundgroup/urltest.go` | url-test 类型实现 |
| `adapter/outboundgroup/fallback.go` | fallback 类型实现 |
| `adapter/outboundgroup/loadbalance.go` | load-balance 类型实现 |

## 代理组类型

| 类型 | 说明 | 源码位置 |
|------|------|----------|
| [[selector]] | 手动选择 | `selector.go` |
| [[url-test]] | 自动选优 | `urltest.go` |
| [[fallback]] | 故障转移 | `fallback.go` |
| [[load-balance]] | 负载均衡 | `loadbalance.go` |

## YAML 配置示例

### 基础结构

```yaml
proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - "节点A"
      - "节点B"
```

### 使用 provider

```yaml
proxy-groups:
  - name: "自动选择"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    use:
      - provider1
```

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

## 配置字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 代理组名称 |
| `type` | string | 代理组类型 |
| `proxies` | []string | 节点列表 |
| `use` | []string | 引用的 provider |
| `url` | string | 测试 URL |
| `interval` | int | 测试间隔（秒） |
| `timeout` | int | 测试超时（毫秒） |
| `lazy` | bool | 懒加载模式 |
| `filter` | string | 节点过滤正则 |
| `hidden` | bool | 面板隐藏 |
| `icon` | string | 面板图标 |

## 相关文档

- [[selector]] - select 代理组
- [[url-test]] - url-test 代理组
- [[fallback]] - fallback 代理组
- [[load-balance]] - load-balance 代理组
- [[config]] - 代理组配置详解
- [[../provider/overview|Provider 概述]]
- [[../../client/mihomo-proxy-groups|代理组概念]]