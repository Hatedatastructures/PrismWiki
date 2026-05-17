---
title: "Proxy Provider"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/provider/proxy.go"
tags: [mihomo, provider, proxy-provider, 代理提供者, 订阅]
created: 2026-05-17
updated: 2026-05-17
related: [overview, rule-provider, health-check, override]
---

# Proxy Provider

**类别**: Mihomo 代理提供者

## 概述

Proxy Provider（代理提供者）从远程服务器获取代理节点列表。支持 HTTP 订阅、本地文件、Base64 解码等多种格式。

## 基础配置

### HTTP 类型

```yaml
proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    interval: 3600
    path: ./proxy/my-provider.yaml
```

### File 类型

```yaml
proxy-providers:
  local-provider:
    type: file
    path: ./proxy/local.yaml
```

### 完整配置

```yaml
proxy-providers:
  my-provider:
    type: http
    url: "https://example.com/api/v1/client/subscribe?token=xxx"
    interval: 3600
    path: ./proxy/my-provider.yaml
    header:
      User-Agent:
        - "ClashMeta"
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
    filter: "香港|HK|台湾|TW"
    exclude-filter: "过期|网址|流量"
    exclude-type: "ss|ssr"
```

## 配置参数

### type

```yaml
type: http
```

| 值 | 说明 |
|-----|------|
| http | HTTP/HTTPS 远程订阅 |
| file | 本地文件 |

### url

```yaml
url: "https://example.com/proxies.yaml"
```

远程订阅地址（仅 HTTP 类型）。

### interval

```yaml
interval: 3600
```

更新间隔（秒）：

| 推荐值 | 场景 |
|--------|------|
| 300 | 高频更新 |
| 3600 | 常规更新 |
| 86400 | 低频更新 |

### path

```yaml
path: ./proxy/my-provider.yaml
```

资源缓存路径。下载后存储在此路径。

### header

```yaml
header:
  User-Agent:
    - "ClashMeta"
  Authorization:
    - "Bearer token"
```

HTTP 请求头自定义。

### health-check

```yaml
health-check:
  enable: true
  url: https://www.gstatic.com/generate_204
  interval: 300
  timeout: 5000
  expected-status: 204
```

健康检查配置，自动测试节点可用性。

### filter

```yaml
filter: "香港|HK|台湾|TW|日本|JP"
```

节点过滤正则表达式，仅保留匹配的节点。

### exclude-filter

```yaml
exclude-filter: "过期|网址|官网|流量"
```

排除节点正则表达式，过滤不需要的节点。

### exclude-type

```yaml
exclude-type: "ss|ssr|vmess"
```

排除特定协议类型。

### override

```yaml
override:
  additional-prefix: "[Provider]"
  additional-suffix: ""
```

节点名称覆盖配置。

## 支持的格式

### Clash YAML 格式

```yaml
proxies:
  - name: "节点1"
    type: ss
    server: server.com
    port: 443
    cipher: aes-128-gcm
    password: "password"
```

### Clash JSON 格式

```json
{
  "proxies": [
    {
      "name": "节点1",
      "type": "ss",
      "server": "server.com",
      "port": 443
    }
  ]
}
```

### Base64 订阅格式

自动解码 Base64 编码的订阅链接（如 V2Ray 订阅）。

## 配置示例

### 基础订阅

```yaml
proxy-providers:
  provider-1:
    type: http
    url: "https://example.com/proxies.yaml"
    interval: 3600
    path: ./provider/provider-1.yaml

proxy-groups:
  - name: "auto"
    type: url-test
    use:
      - provider-1
    url: "https://www.gstatic.com/generate_204"
    interval: 300
```

### 多订阅源

```yaml
proxy-providers:
  provider-1:
    type: http
    url: "https://provider1.com/proxies.yaml"
    interval: 3600
  provider-2:
    type: http
    url: "https://provider2.com/proxies.yaml"
    interval: 3600

proxy-groups:
  - name: "all"
    type: load-balance
    use:
      - provider-1
      - provider-2
```

### 过滤订阅

```yaml
proxy-providers:
  hk-provider:
    type: http
    url: "https://example.com/proxies.yaml"
    interval: 3600
    filter: "香港|HK|Hong Kong"
    health-check:
      enable: true
      interval: 300
```

### 带认证订阅

```yaml
proxy-providers:
  auth-provider:
    type: http
    url: "https://example.com/api/v1/client/subscribe?token=xxx"
    interval: 3600
    header:
      User-Agent:
        - "mihomo/v1.18.0"
      Authorization:
        - "Bearer your-token"
```

### 本地文件

```yaml
proxy-providers:
  local:
    type: file
    path: ./proxy/local.yaml
```

## 与代理组配合

### url-test 代理组

```yaml
proxy-groups:
  - name: "auto"
    type: url-test
    use:
      - my-provider
    url: "https://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
```

### select 代理组

```yaml
proxy-groups:
  - name: "select"
    type: select
    use:
      - my-provider
    proxies:
      - DIRECT
```

### load-balance 代理组

```yaml
proxy-groups:
  - name: "balance"
    type: load-balance
    use:
      - my-provider
    strategy: consistent-hashing
```

## 相关链接

- [[ref/mihomo/provider/overview|Provider 概览]] — 提供者总览
- [[ref/mihomo/provider/rule-provider|Rule Provider]] — 规则提供者
- [[ref/mihomo/provider/health-check|Health Check]] — 健康检查配置
- [[ref/mihomo/provider/override|Override]] — 覆盖配置