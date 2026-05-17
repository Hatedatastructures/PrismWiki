---
title: "启用 NTP"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/ntp"
tags: [mihomo, ntp, enable, 启用, 时间同步]
created: 2026-05-17
updated: 2026-05-17
related: [overview, server]
---

# 启用 NTP

**类别**: Mihomo NTP 配置

## 概述

本文档详细说明如何在 Mihomo 中启用和配置 NTP 时间同步功能。

## 基础启用

### 最小配置

```yaml
ntp:
  enable: true
```

启用后，Mihomo 将使用默认服务器同步时间。

### 完整配置

```yaml
ntp:
  enable: true
  server: time.apple.com
  port: 123
  interval: 3600
```

## 配置参数

### enable

```yaml
enable: true
```

| 值 | 说明 |
|-----|------|
| true | 启用 NTP 同步 |
| false | 禁用 NTP 同步（默认） |

启用后：

1. Mihomo 启动时同步时间
2. 定时同步（按 interval 配置）
3. 记录时钟偏移用于校正

### server

```yaml
server: time.apple.com
```

NTP 服务器地址。

默认值：`time.apple.com`

### port

```yaml
port: 123
```

NTP 服务端口。

默认值：`123`

| 值 | 说明 |
|-----|------|
| 123 | 标准 NTP 端口 |
| 其他 | 自定义端口 |

### interval

```yaml
interval: 3600
```

同步间隔（秒）。

默认值：`3600`（每小时）

| 推荐值 | 场景 |
|--------|------|
| 1800 | 高精度同步 |
| 3600 | 常规同步 |
| 7200 | 低频同步 |

## 启动行为

启用 NTP 后，Mihomo 在启动时：

```
启动流程：
┌─────────────────────────────────────────────┐
│                                             │
│  Mihomo 启动                                │
│      │                                      │
│      ├── 加载配置                            │
│      │                                      │
│      ├── 检查 ntp.enable                    │
│      │                                      │
│      ├── true: 发送 NTP 请求                │
│      │                                      │
│      ├── 接收响应                            │
│      │                                      │
│      ├── 计时钟偏移                          │
│      │                                      │
│      ├── 尝试调整系统时间                    │
│      │   （需要权限）                        │
│      │                                      │
│      └── 继续启动其他模块                    │
│                                             │
└─────────────────────────────────────────────┘
```

## 权限要求

NTP 时间同步可能需要特殊权限：

| 操作 | 权限要求 |
|------|----------|
| 读取时间 | 无需权限 |
| 调整系统时间 | root/管理员 |

### Linux

```bash
# 使用 sudo 运行
sudo mihomo -d /path/to/config

# 或添加能力
sudo setcap CAP_SYS_TIME=+ep /usr/bin/mihomo
```

### Windows

需要管理员权限调整系统时间。

### macOS

需要 root 权限。

## 配置示例

### 基础启用

```yaml
ntp:
  enable: true
```

### 指定服务器

```yaml
ntp:
  enable: true
  server: time.google.com
```

### 高频同步

```yaml
ntp:
  enable: true
  server: time.apple.com
  interval: 1800
```

### 自定义端口

```yaml
ntp:
  enable: true
  server: ntp.example.com
  port: 1234
```

### 完整配置

```yaml
ntp:
  enable: true
  server: time.apple.com
  port: 123
  interval: 3600
```

## 故障处理

### 时间偏差过大

如果时间偏差超过阈值：

- TLS 证书验证可能失败
- 时间敏感协议可能报错

解决方案：

1. 手动同步系统时间
2. 检查 NTP 配置
3. 使用可靠 NTP 服务器

### 无法连接 NTP 服务器

检查项：

1. NTP 服务器是否可达
2. 网络连接是否正常
3. 端口是否正确

### 权限不足

如果无法调整系统时间：

- Mihomo 会记录偏移值
- 内部计算会使用校正时间
- 建议以管理员/root 权限运行

## 相关链接

- [[ref/mihomo/ntp/overview|NTP 概览]] — NTP 功能总览
- [[ref/mihomo/ntp/server|NTP 服务器]] — 服务器配置详解