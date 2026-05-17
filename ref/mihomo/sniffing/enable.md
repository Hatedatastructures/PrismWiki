---
title: "启用嗅探"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/component/sniffer"
tags: [mihomo, sniffing, enable, 启用, 配置]
created: 2026-05-17
updated: 2026-05-17
related: [sniffing-types, ports]
---

# 启用嗅探

**类别**: Mihomo 嗅探配置

## 概述

本文档详细说明如何在 Mihomo 中启用和配置嗅探功能。

## 基础启用

### 最小配置

```yaml
sniffer:
  enable: true
```

启用后，Mihomo 将自动嗅探所有连接的协议类型。

### 完整配置

```yaml
sniffer:
  enable: true
  parse-pure-ip: true
  sniff:
    HTTP:
      ports: [80, 8080-8090]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
    STUN:
      ports: [3478, 5349]
  force-domain:
    - "+.google.com"
    - "+.github.com"
  skip-domain:
    - "miejue.org"
  skip-ip:
    - "192.168.0.0/16"
    - "10.0.0.0/8"
```

## enable

```yaml
enable: true
```

| 值 | 说明 |
|-----|------|
| true | 启用嗅探功能 |
| false | 禁用嗅探功能（默认） |

嗅探启用后：

1. 所有入站连接都会进行协议检测
2. 检测结果用于路由规则匹配
3. 可覆盖原始目标地址

## parse-pure-ip

```yaml
parse-pure-ip: true
```

| 值 | 说明 |
|-----|------|
| true | 嗅探纯 IP 连接 |
| false | 跳过纯 IP 连接嗅探 |

行为说明：

- **true**：对 IP 直连目标也进行嗅探
- **false**：跳过无域名的 IP 连接

适用场景：

- 启用：需要识别 IP 访问的实际协议
- 禁用：减少不必要开销，IP 访问通常无协议特征

## force-domain

强制对特定域名进行嗅探：

```yaml
force-domain:
  - "+.google.com"    # 所有 google.com 子域名
  - "+.github.com"    # 所有 github.com 子域名
  - "example.org"     # 精确匹配
```

匹配模式：

| 模式 | 示例 | 匹配范围 |
|------|------|----------|
| 精确匹配 | `example.org` | 仅匹配 example.org |
| 前缀通配 | `+.example.org` | 匹配所有 *.example.org |

## skip-domain

跳过特定域名的嗅探：

```yaml
skip-domain:
  - "miejue.org"      # 已知无需嗅探
  - "localhost"       # 本地地址
```

适用场景：

- 已知协议的域名
- 嗅探结果不准确的情况
- 减少不必要的处理

## force-ip

强制对特定 IP 进行嗅探：

```yaml
force-ip:
  - "8.8.8.8"          # 精确 IP
  - "1.1.1.0/24"       # CIDR 网段
```

## skip-ip

跳过特定 IP 的嗅探：

```yaml
skip-ip:
  - "192.168.0.0/16"   # 私有网段
  - "10.0.0.0/8"       # 私有网段
  - "172.16.0.0/12"    # 私有网段
  - "127.0.0.0/8"      # 本地回环
```

推荐配置：跳过私有 IP 可减少不必要开销。

## 配置示例

### 基础配置

```yaml
sniffer:
  enable: true
  parse-pure-ip: false
  sniff:
    HTTP:
      ports: [80, 8080-8090]
    TLS:
      ports: [443, 8443]
```

### 完整配置

```yaml
sniffer:
  enable: true
  parse-pure-ip: true
  sniff:
    HTTP:
      ports: [80, 8080-8090]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
    STUN:
      ports: [3478, 5349]
  force-domain:
    - "+.google.com"
  skip-domain:
    - "miejue.org"
  skip-ip:
    - "192.168.0.0/16"
    - "10.0.0.0/8"
```

### 隐私保护配置

```yaml
sniffer:
  enable: true
  parse-pure-ip: false
  sniff:
    TLS:
      ports: [443]
  skip-domain:
    - "+.local"
    - "+.lan"
  skip-ip:
    - "192.168.0.0/16"
    - "10.0.0.0/8"
    - "172.16.0.0/12"
```

## 相关链接

- [[ref/mihomo/sniffing/overview|Sniffing 概览]] — 嗅探功能总览
- [[ref/mihomo/sniffing/sniffing-types|嗅探类型]] — 协议嗅探详解
- [[ref/mihomo/sniffing/ports|端口配置]] — 嗅探端口配置