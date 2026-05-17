---
title: "嗅探端口配置"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/component/sniffer"
tags: [mihomo, sniffing, ports, 端口, 配置]
created: 2026-05-17
updated: 2026-05-17
related: [enable, sniffing-types]
---

# 嗅探端口配置

**类别**: Mihomo 嗅探配置

## 概述

本文档详细说明嗅探功能的端口配置，包括端口范围语法和常见配置示例。

## 端口配置语法

### 单个端口

```yaml
ports: [80]
```

指定单个端口号。

### 多个端口

```yaml
ports: [80, 443, 8080]
```

列出多个端口号，用逗号分隔。

### 端口范围

```yaml
ports: [8080-8090]
```

使用 `-` 表示端口范围，包含起始和结束端口。

### 混合配置

```yaml
ports: [80, 443, 8080-8090, 9000-9100]
```

混合使用单个端口和端口范围。

### 字符串格式

```yaml
ports: "80,443,8080-8090"
```

也支持字符串格式，用逗号分隔。

## 各协议默认端口

| 协议 | 默认端口 | 常见替代端口 |
|------|----------|--------------|
| HTTP | 80 | 8080, 8888, 9000 |
| HTTPS/TLS | 443 | 8443, 9443 |
| QUIC | 443 | 8443 |
| DNS | 53 | 5353 |
| STUN | 3478 | 5349 |

## HTTP 端口配置

### 基础配置

```yaml
sniffer:
  sniff:
    HTTP:
      ports: [80]
```

### 扩展配置

```yaml
sniffer:
  sniff:
    HTTP:
      ports: [80, 8080, 8081, 8888, 9000-9999]
```

### 常见 Web 端口

```yaml
ports: [80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 443, 8000-9099]
```

## TLS 端口配置

### 基础配置

```yaml
sniffer:
  sniff:
    TLS:
      ports: [443]
```

### 扩展配置

```yaml
sniffer:
  sniff:
    TLS:
      ports: [443, 8443, 9443, 10443]
```

### HTTPS 常见端口

```yaml
ports: [443, 8443, 9443, 2443, 3443, 4443, 5443, 6443, 7443]
```

## QUIC 端口配置

### 基础配置

```yaml
sniffer:
  sniff:
    QUIC:
      ports: [443]
```

QUIC 通常与 HTTPS 共用 443 端口。

### 扩展配置

```yaml
sniffer:
  sniff:
    QUIC:
      ports: [443, 8443]
```

## DNS 端口配置

### 标准配置

```yaml
sniffer:
  sniff:
    DNS:
      ports: [53]
```

### DoH/DoT 端口

DNS over HTTPS 和 DNS over TLS 不是通过端口嗅探，而是通过协议特征。

## STUN 端口配置

### 标准配置

```yaml
sniffer:
  sniff:
    STUN:
      ports: [3478, 5349]
```

| 端口 | 用途 |
|------|------|
| 3478 | STUN 标准端口 |
| 5349 | STUN over TLS |

## 配置示例

### 最小配置

```yaml
sniffer:
  enable: true
  sniff:
    HTTP:
      ports: [80]
    TLS:
      ports: [443]
```

### 常用配置

```yaml
sniffer:
  enable: true
  sniff:
    HTTP:
      ports: [80, 8080-8090]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
```

### 全面嗅探

```yaml
sniffer:
  enable: true
  parse-pure-ip: true
  sniff:
    HTTP:
      ports: [80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 8000-9999]
      override-destination: true
    TLS:
      ports: [443, 8443, 9443]
    QUIC:
      ports: [443, 8443]
    STUN:
      ports: [3478, 5349]
    DNS:
      ports: [53]
```

### 仅 HTTPS 嗅探

```yaml
sniffer:
  enable: true
  sniff:
    TLS:
      ports: [443]
    QUIC:
      ports: [443]
```

### 端口范围语法示例

```yaml
# 有效配置示例
ports: [80]                    # 单端口
ports: [80, 443]              # 多端口
ports: [8080-8090]            # 范围
ports: [80, 443, 8080-8090]   # 混合
ports: "80,443,8080-8090"     # 字符串格式
```

## 性能考量

端口配置影响嗅探性能：

| 配置 | 性能影响 |
|------|----------|
| 端口少 | 嗅探范围小，性能好 |
| 端口多 | 嗅探范围大，CPU 开销增加 |
| 范围大 | 可能影响连接延迟 |

建议：仅配置常用端口，避免不必要的嗅探。

## 相关链接

- [[ref/mihomo/sniffing/overview|Sniffing 概览]] — 嗅探功能总览
- [[ref/mihomo/sniffing/enable|启用嗅探]] — 启用配置详解
- [[ref/mihomo/sniffing/sniffing-types|嗅探类型]] — 协议嗅探详解