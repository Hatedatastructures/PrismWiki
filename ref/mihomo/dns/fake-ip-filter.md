---
title: "mihomo DNS fake-ip-filter 配置"
category: "mihomo"
type: ref
layer: ref
module: "dns"
source: "mihomo-Meta"
tags: [mihomo, dns, fake-ip-filter, fake-ip]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo DNS fake-ip-filter 配置

`fake-ip-filter` 定义不使用 fake-ip 的域名列表。

## 源码位置

| 文件 | 描述 |
|------|------|
| `dns/enhancer.go` | ResolverEnhancer 实现 |
| `component/fakeip/` | fake-ip 模块 |

## YAML 配置

```yaml
dns:
  enhanced-mode: fake-ip
  fake-ip-filter:
    - "*.lan"
    - "*.local"
    - "*.localhost"
    - "dns.msftncsi.com"
    - "+.apple.com"
    - "*.ntp.org.cn"
```

## 过滤规则语法

| 语法 | 说明 | 示例 |
|------|------|------|
| `*` | 单层通配符 | `*.lan` |
| `+` | 多层通配符 | `+.ntp.org` |
| 精确域名 | 精确匹配 | `localhost` |

## 常用过滤规则

### 局域网域名

```yaml
fake-ip-filter:
  - "*.lan"
  - "*.local"
  - "*.localhost"
  - "*.localdomain"
  - "*.internal"
  - "*.private"
```

### 网络检测域名

```yaml
fake-ip-filter:
  # Windows
  - "dns.msftncsi.com"
  - "www.msftncsi.com"
  - "+.msftconnecttest.com"
  
  # macOS/iOS
  - "captive.apple.com"
  - "+.apple.com"
  
  # Android
  - "connectivitycheck.gstatic.com"
```

### 时间同步域名

```yaml
fake-ip-filter:
  - "*.ntp.org.cn"
  - "time.*.com"
  - "time.*.gov"
  - "+.ntp.org"
```

### STUN/CDN 服务

```yaml
fake-ip-filter:
  - "+.stun.*"
  - "+.stun.*.*"
  - "+.stun.*.*.*"
```

### 金融服务

```yaml
fake-ip-filter:
  - "+.bank"
  - "+.pay"
```

## 完整配置示例

```yaml
dns:
  enable: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  
  fake-ip-filter:
    # 局域网
    - "*.lan"
    - "*.local"
    - "*.localhost"
    - "*.localdomain"
    
    # Windows 网络检测
    - "dns.msftncsi.com"
    - "www.msftncsi.com"
    - "+.msftconnecttest.com"
    
    # macOS/iOS 网络检测
    - "captive.apple.com"
    - "+.apple.com"
    
    # 时间同步
    - "*.ntp.org.cn"
    - "time.*.com"
    - "+.ntp.org"
    
    # STUN 服务
    - "+.stun.*"
    - "+.stun.*.*"
    
    # 金融
    - "+.bank"
    - "+.pay"
    
  nameserver:
    - https://dns.alidns.com/dns-query
```

## 过滤原因说明

| 类别 | 原因 |
|------|------|
| 局域网域名 | 无法通过代理访问 |
| 网络检测 | 系统需要真实响应 |
| 时间同步 | NTP 需要准确 IP |
| STUN/CDN | 需要 IP 判断地理位置 |
| 金融支付 | 安全敏感，需要可靠解析 |

## 相关文档

- [[overview]] - DNS 配置概述
- [[enhanced-mode]] - DNS 增强模式
- [[ref/mihomo/dns/servers|servers]] - DNS 服务器配置