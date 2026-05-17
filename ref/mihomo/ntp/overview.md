---
title: "NTP 概览"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/ntp"
tags: [mihomo, ntp, 时间同步, 系统时间]
created: 2026-05-17
updated: 2026-05-17
related: [enable, server]
---

# NTP 概览

**类别**: Mihomo NTP 时间同步参考

## 概述

NTP（Network Time Protocol）配置允许 Mihomo 与远程服务器同步系统时间。准确的时间对于 TLS 连接、证书验证、时间敏感协议非常重要。

## 配置位置

```yaml
ntp:
  enable: true
  server: time.apple.com
  port: 123
  interval: 3600
```

## NTP 工作原理

```
NTP 同步流程：
┌─────────────────────────────────────────────┐
│                                             │
│  Mihomo                                     │
│      │                                      │
│      │ 启动时/定时发送 NTP 请求              │
│      ▼                                      │
│  NTP 服务器                                  │
│      │                                      │
│      │ 返回当前时间                          │
│      ▼                                      │
│  Mihomo                                     │
│      │                                      │
│      │ 计算时钟偏移                          │
│      │                                      │
│      ├── 更新系统时间                        │
│      │   （需要权限）                        │
│      │                                      │
│      └── 记录偏移值                          │
│          用于时间校正                        │
│                                             │
└─────────────────────────────────────────────┘
```

## 核心配置项

| 配置项 | 类型 | 说明 |
|--------|------|------|
| enable | bool | 启用 NTP 同步 |
| server | string | NTP 服务器地址 |
| port | int | NTP 端口（默认 123） |
| interval | int | 同步间隔（秒） |

## 时间的重要性

### TLS 连接

TLS 证书有时间有效期：

- 时间偏差过大可能导致证书验证失败
- 证书过期检测依赖系统时间

### 时间敏感协议

部分协议依赖时间：

- VMess：时间戳校验
- VLESS：可能包含时间校验
- Shadowsocks：部分实现有时间检查

### 日志记录

准确时间用于：

- 日志时间戳
- 连接统计
- 调试排查

## 配置示例

### 基础配置

```yaml
ntp:
  enable: true
  server: time.apple.com
```

### 完整配置

```yaml
ntp:
  enable: true
  server: time.apple.com
  port: 123
  interval: 3600
```

### 多服务器配置

```yaml
ntp:
  enable: true
  server: time.google.com
  port: 123
  interval: 1800
```

## 常用 NTP 服务器

| 服务器 | 所属 | 说明 |
|--------|------|------|
| time.apple.com | Apple | 稳定可靠 |
| time.google.com | Google | 全球服务 |
| time.windows.com | Microsoft | Windows 默认 |
| pool.ntp.org | NTP Pool | 公共池 |
| cn.pool.ntp.org | NTP Pool | 中国区域 |

## 相关链接

- [[ref/mihomo/ntp/enable|启用 NTP]] — NTP 启用配置详解
- [[ref/mihomo/ntp/server|NTP 服务器]] — 服务器配置详解