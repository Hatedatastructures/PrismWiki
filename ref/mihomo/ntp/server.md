---
title: "NTP 服务器"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/ntp"
tags: [mihomo, ntp, server, 服务器, 时间同步]
created: 2026-05-17
updated: 2026-05-17
related: [overview, enable]
---

# NTP 服务器

**类别**: Mihomo NTP 配置

## 概述

本文档详细说明 NTP 服务器配置，包括常用服务器列表和选择建议。

## 配置参数

### server

```yaml
server: time.apple.com
```

NTP 服务器地址。

| 格式 | 示例 | 说明 |
|------|------|------|
| 域名 | time.apple.com | 推荐 |
| IP | 17.253.114.253 | 可用 |

### port

```yaml
port: 123
```

NTP 端口。

标准 NTP 端口为 123。

## 常用 NTP 服务器

### Apple

```yaml
ntp:
  server: time.apple.com
```

| 服务器 | 地址 | 说明 |
|--------|------|------|
| time.apple.com | Apple NTP | 稳定可靠 |
| time1.apple.com | 17.253.114.253 | 区域服务器 |
| time2.apple.com | 17.253.114.254 | 区域服务器 |

### Google

```yaml
ntp:
  server: time.google.com
```

| 服务器 | 地址 | 说明 |
|--------|------|------|
| time.google.com | Google NTP | 全球服务 |
| time1.google.com | 216.239.35.0 | 区域服务器 |
| time2.google.com | 216.239.35.4 | 区域服务器 |
| time3.google.com | 216.239.35.8 | 区域服务器 |
| time4.google.com | 216.239.35.12 | 区域服务器 |

### Microsoft

```yaml
ntp:
  server: time.windows.com
```

Windows 系统默认 NTP 服务器。

### NTP Pool

```yaml
ntp:
  server: pool.ntp.org
```

公共 NTP 服务器池。

| 服务器 | 说明 |
|--------|------|
| pool.ntp.org | 全球池 |
| cn.pool.ntp.org | 中国区域 |
| asia.pool.ntp.org | 亚洲区域 |
| europe.pool.ntp.org | 欧洲区域 |
| north-america.pool.ntp.org | 北美区域 |

### Cloudflare

```yaml
ntp:
  server: time.cloudflare.com
```

Cloudflare 提供的 NTP 服务。

### 其他

```yaml
ntp:
  server: ntp.ubuntu.com
```

| 服务器 | 所属 |
|--------|------|
| ntp.ubuntu.com | Ubuntu |
| clock.isc.org | ISC |
| ntp.nict.jp | NICT (日本) |

## 服务器选择建议

### 按地理位置选择

选择距离较近的服务器：

| 地区 | 推荐服务器 |
|------|------------|
| 中国 | cn.pool.ntp.org, time.apple.com |
| 日本 | ntp.nict.jp, time.apple.com |
| 美国 | time.google.com, pool.ntp.org |
| 欧洲 | europe.pool.ntp.org, time.google.com |

### 按可靠性选择

| 服务器 | 可用性 | 响应时间 |
|--------|--------|----------|
| time.apple.com | 高 | 中等 |
| time.google.com | 高 | 中等 |
| pool.ntp.org | 中 | 变化 |
| time.cloudflare.com | 高 | 低 |

### 按用途选择

| 场景 | 推荐 |
|------|------|
| 一般使用 | time.apple.com |
| 高精度 | time.google.com |
| 中国本地 | cn.pool.ntp.org |
| 企业内网 | 自建 NTP |

## NTP 协议

### NTP 包格式

```
NTP 包结构：
┌────────────────────────────────────────────┐
│ Flags: 1 byte                              │
│   LI (Leap Indicator): 2 bits              │
│   VN (Version Number): 3 bits              │
│   Mode: 3 bits                             │
│ Stratum: 1 byte                            │
│ Poll: 1 byte                               │
│ Precision: 1 byte                          │
│ Root Delay: 4 bytes                        │
│ Root Dispersion: 4 bytes                   │
│ Reference ID: 4 bytes                      │
│ Reference Timestamp: 8 bytes               │
│ Origin Timestamp: 8 bytes                  │
│ Receive Timestamp: 8 bytes                 │
│ Transmit Timestamp: 8 bytes                │
└────────────────────────────────────────────┘
```

### 时间计算

时钟偏移计算：

```
offset = ((T2 - T1) + (T3 - T4)) / 2

T1: 客户端发送时间
T2: 服务器接收时间
T3: 服务器发送时间
T4: 客户端接收时间
```

## 配置示例

### Apple 时间服务器

```yaml
ntp:
  enable: true
  server: time.apple.com
  port: 123
  interval: 3600
```

### Google 时间服务器

```yaml
ntp:
  enable: true
  server: time.google.com
  port: 123
  interval: 3600
```

### 中国区域 NTP Pool

```yaml
ntp:
  enable: true
  server: cn.pool.ntp.org
  port: 123
  interval: 3600
```

### Cloudflare 时间服务器

```yaml
ntp:
  enable: true
  server: time.cloudflare.com
  port: 123
  interval: 1800
```

### 自定义 NTP 服务器

```yaml
ntp:
  enable: true
  server: ntp.example.com
  port: 123
  interval: 3600
```

## 排错指南

### 测试 NTP 连接

```bash
# 使用 ntpdate 测试
ntpdate -q time.apple.com

# 使用 ntpq 测试
ntpq -p time.apple.com
```

### 检查时间偏差

```bash
# Linux
timedatectl status

# macOS
systemsetup -gettime
```

### 验证配置

检查 Mihomo 日志：

```
[NTP] sync time from time.apple.com:123
[NTP] offset: +123ms
```

## 相关链接

- [[ref/mihomo/ntp/overview|NTP 概览]] — NTP 功能总览
- [[ref/mihomo/ntp/enable|启用 NTP]] — NTP 启用配置详解