---
title: "Provider 概览"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/provider"
tags: [mihomo, provider, 代理提供者, 规则提供者, 外部资源]
created: 2026-05-17
updated: 2026-05-17
related: [proxy-provider, rule-provider, health-check, override]
---

# Provider 概览

**类别**: Mihomo 外部资源提供者参考

## 概述

Provider（提供者）是 Mihomo 的外部资源管理机制，允许从远程服务器获取代理节点和规则列表。Provider 实现了代理和规则的动态更新，无需修改配置文件即可更新资源。

### Provider 类型

| 类型 | 说明 |
|------|------|
| [[ref/mihomo/provider/proxy-provider|Proxy Provider]] | 代理节点提供者 |
| [[ref/mihomo/provider/rule-provider|Rule Provider]] | 规则列表提供者 |

## 配置位置

Provider 配置在 `proxy-providers` 和 `rule-providers` 配置块中：

```yaml
proxy-providers:
  provider-name:
    type: http
    url: "https://example.com/proxies.yaml"
    interval: 3600
    # ...

rule-providers:
  rule-name:
    type: http
    url: "https://example.com/rules.yaml"
    interval: 86400
    # ...
```

## Provider 工作流程

```
Provider 工作流程：
┌─────────────────────────────────────────────┐
│                                             │
│  Mihomo 启动                                │
│      │                                      │
│      ├── 加载 Provider 配置                 │
│      │                                      │
│      ├── 首次下载资源                        │
│      │                                      │
│      ├── 解析代理/规则                       │
│      │                                      │
│      ├── 应用到代理组/规则                   │
│      │                                      │
│      └── 定时更新（interval）               │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ 远程服务器                            │   │
│  │   - YAML/JSON 格式                   │   │
│  │   - Base64 编码                      │   │
│  │   - 订阅链接                          │   │
│  └─────────────────────────────────────┘   │
│                                             │
└─────────────────────────────────────────────┘
```

## 通用配置参数

### type

```yaml
type: http
```

Provider 类型：

| 值 | 说明 |
|-----|------|
| http | HTTP/HTTPS 远程资源 |
| file | 本地文件 |

### path

```yaml
path: ./provider/proxies.yaml
```

资源存储路径。

### interval

```yaml
interval: 3600
```

更新间隔（秒）。定期检查远程资源更新。

### health-check

```yaml
health-check:
  enable: true
  url: https://www.gstatic.com/generate_204
  interval: 300
```

健康检查配置（仅 Proxy Provider）。

### filter

```yaml
filter: "香港|HK|Taiwan"
```

节点过滤正则表达式。

### exclude-filter

```yaml
exclude-filter: "过期|过期|网址"
```

排除节点正则表达式。

## Provider 格式

### YAML 格式

```yaml
proxies:
  - name: "node-1"
    type: ss
    server: server1.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
  - name: "node-2"
    type: trojan
    server: server2.com
    port: 443
    password: "password"
```

### JSON 格式

```json
{
  "proxies": [
    {
      "name": "node-1",
      "type": "ss",
      "server": "server1.com",
      "port": 443,
      "cipher": "aes-128-gcm",
      "password": "password"
    }
  ]
}
```

### Base64 格式

Base64 编码的订阅链接也支持解析。

## 与代理组配合

```yaml
proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    interval: 3600

proxy-groups:
  - name: "auto"
    type: url-test
    use:
      - my-provider
    url: "https://www.gstatic.com/generate_204"
    interval: 300
```

## 与规则配合

```yaml
rule-providers:
  reject:
    type: http
    url: "https://example.com/reject.yaml"
    interval: 86400

rules:
  - RULE-SET,reject,REJECT
```

## 相关链接

- [[ref/mihomo/provider/proxy-provider|Proxy Provider]] — 代理提供者详解
- [[ref/mihomo/provider/rule-provider|Rule Provider]] — 规则提供者详解
- [[ref/mihomo/provider/health-check|Health Check]] — 健康检查配置
- [[ref/mihomo/provider/override|Override]] — 覆盖配置