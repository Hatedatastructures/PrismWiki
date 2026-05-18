---
title: "Rule Provider"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/provider/rule.go"
tags: [mihomo, provider, rule-provider, 规则提供者, 外部规则]
created: 2026-05-17
updated: 2026-05-17
related: [overview, proxy-provider]
---

# Rule Provider

**类别**: Mihomo 规则提供者

## 概述

Rule Provider（规则提供者）从远程服务器或本地文件获取规则列表。支持域名规则、IP 规则、进程规则等多种类型。

## 基础配置

### HTTP 类型

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://example.com/reject.yaml"
    interval: 86400
    path: ./rule/reject.yaml
```

### File 类型

```yaml
rule-providers:
  local-rules:
    type: file
    behavior: classical
    path: ./rule/local.yaml
```

### 完整配置

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt"
    interval: 86400
    path: ./rule/reject.yaml
    format: yaml
```

## 配置参数

### type

```yaml
type: http
```

| 值 | 说明 |
|-----|------|
| http | HTTP/HTTPS 远程资源 |
| file | 本地文件 |

### behavior

```yaml
behavior: domain
```

规则行为类型：

| 值 | 说明 | 规则格式 |
|-----|------|----------|
| domain | 域名规则 | DOMAIN-SUFFIX |
| ipcidr | IP 规则 | IP-CIDR |
| classical | 混合规则 | 完整规则格式 |

### url

```yaml
url: "https://example.com/rules.yaml"
```

远程规则地址（仅 HTTP 类型）。

### interval

```yaml
interval: 86400
```

更新间隔（秒）：

| 推荐值 | 场景 |
|--------|------|
| 86400 | 每日更新 |
| 43200 | 每12小时 |
| 604800 | 每周更新 |

### path

```yaml
path: ./rule/reject.yaml
```

规则缓存路径。

### format

```yaml
format: yaml
```

规则格式：

| 值 | 说明 |
|-----|------|
| yaml | YAML 格式 |
| text | 纯文本格式 |
| mrs | Mihomo 二进制格式 |

### size-limit

```yaml
size-limit: 5242880
```

文件大小限制（字节），默认 5MB。

## 规则行为类型

### domain 行为

```yaml
behavior: domain
```

仅包含域名规则：

```yaml
payload:
  - "+.google.com"
  - "+.github.com"
  - "example.org"
```

自动转换为 `DOMAIN-SUFFIX` 规则。

### ipcidr 行为

```yaml
behavior: ipcidr
```

仅包含 IP 规则：

```yaml
payload:
  - "8.8.8.8/32"
  - "1.1.1.0/24"
  - "192.168.0.0/16"
```

自动转换为 `IP-CIDR` 规则。

### classical 行为

```yaml
behavior: classical
```

完整规则格式：

```yaml
payload:
  - DOMAIN-SUFFIX,google.com,PROXY
  - IP-CIDR,8.8.8.8/32,PROXY,no-resolve
  - PROCESS-NAME,chrome.exe,PROXY
```

支持所有规则类型。

## 规则格式

### YAML 格式

```yaml
payload:
  - "+.google.com"
  - "+.github.com"
```

### Text 格式

```
+.google.com
+.github.com
example.org
```

### Classical YAML 格式

```yaml
payload:
  - DOMAIN-SUFFIX,google.com,PROXY
  - DOMAIN-KEYWORD,github,PROXY
  - IP-CIDR,8.8.8.8/32,PROXY,no-resolve
  - SRC-IP-CIDR,192.168.0.0/16,DIRECT
  - PROCESS-NAME,chrome.exe,PROXY
  - RULE-SET,other-provider,PROXY
  - GEOSITE,cn,DIRECT
  - GEOIP,cn,DIRECT
```

## 在规则中使用

### RULE-SET 规则

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://example.com/reject.yaml"

rules:
  - RULE-SET,reject,REJECT
  - MATCH,PROXY
```

### 多规则集

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://example.com/reject.yaml"
  proxy:
    type: http
    behavior: domain
    url: "https://example.com/proxy.yaml"
  direct:
    type: http
    behavior: domain
    url: "https://example.com/direct.yaml"

rules:
  - RULE-SET,reject,REJECT
  - RULE-SET,proxy,PROXY
  - RULE-SET,direct,DIRECT
  - MATCH,PROXY
```

## 配置示例

### 广告拦截

```yaml
rule-providers:
  advertising:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt"
    interval: 86400
    path: ./rule/advertising.yaml

rules:
  - RULE-SET,advertising,REJECT
```

### 中国直连

```yaml
rule-providers:
  cn-domain:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/direct-cdn-domains.txt"
    interval: 86400
  cn-ip:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cncidr.txt"
    interval: 86400

rules:
  - RULE-SET,cn-domain,DIRECT
  - RULE-SET,cn-ip,DIRECT
```

### 代理域名

```yaml
rule-providers:
  proxy-domains:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    interval: 86400

rules:
  - RULE-SET,proxy-domains,PROXY
```

### 本地规则文件

```yaml
rule-providers:
  my-rules:
    type: file
    behavior: classical
    path: ./rule/my-rules.yaml

rules:
  - RULE-SET,my-rules,PROXY
```

## 常用规则集来源

| 来源 | URL | behavior |
|------|-----|----------|
| Loyalsoldier | `gh/Loyalsoldier/clash-rules` | domain/ipcidr |
| HackDeveloper | `gh/HackDeveloper0/clash-rules` | domain |
| blackmatrix7 | `gh/blackmatrix7/ios_rule_script` | classical |

## 相关链接

- [[ref/mihomo/provider/overview|Provider 概览]] — 提供者总览
- [[ref/mihomo/provider/proxy-provider|Proxy Provider]] — 代理提供者
- [[ref/mihomo/rules/overview|rules]] — 规则配置参考