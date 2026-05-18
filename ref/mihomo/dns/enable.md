---
title: "mihomo DNS enable 配置"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, enable, 启用]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS enable 配置

`enable` 控制是否启用 mihomo 内置 DNS 模块。

## YAML 配置

```yaml
dns:
  enable: true  # 启用 DNS 模块
```

```yaml
dns:
  enable: false  # 禁用 DNS 模块
```

## 配置说明

| 值 | 说明 |
|---|------|
| `true` | 启用内置 DNS 模块 |
| `false` | 禁用内置 DNS，使用系统 DNS |

## 默认值

默认值为 `false`，需要显式设置为 `true` 才能启用 DNS 模块。

## 使用场景

### 启用 DNS 模块（推荐）

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  nameserver:
    - https://dns.alidns.com/dns-query
  fallback:
    - https://dns.google/dns-query
```

启用 DNS 模块的优点：
- 防止 DNS 泄露
- 处理 DNS 污染
- 支持 fake-ip 模式
- 实现 DNS 分流

### 禁用 DNS 模块

```yaml
dns:
  enable: false
```

禁用 DNS 模块的场景：
- 使用外部 DNS 服务
- 系统级 DNS 已配置
- 不需要 DNS 功能

## 相关配置项

当 `enable: true` 时，以下配置项生效：

| 配置项 | 说明 |
|--------|------|
| [[ref/mihomo/dns/listen|listen]] | DNS 监听地址 |
| [[enhanced-mode]] | DNS 增强模式 |
| [[ref/mihomo/dns/servers|servers]] | DNS 服务器列表 |
| [[ref/mihomo/dns/fallback|fallback]] | fallback DNS |
| [[ref/mihomo/dns/fake-ip-filter|fake-ip-filter]] | fake-ip 过滤列表 |
| [[ref/mihomo/dns/nameserver-policy|nameserver-policy]] | DNS 策略 |

## 相关文档

- [[overview]] - DNS 配置概述
- [[enhanced-mode]] - DNS 增强模式
- [[ref/mihomo/dns/servers|servers]] - DNS 服务器配置