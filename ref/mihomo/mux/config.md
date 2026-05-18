---
title: "Mux 配置参数"
category: "mihomo"
type: ref
module: ref/mihomo
source: "mihomo-Meta/adapter/outbound/singmux.go"
tags: [mihomo, mux, config, 配置, smux, brutal]
created: 2026-05-17
updated: 2026-05-17
related: [singmux, smux, yamux]
layer: ref
---

# Mux 配置参数

**类别**: Mihomo 配置参考

## 概述

本文档详细说明 Mihomo 多路复用（Mux）的所有配置参数，包括 Sing-Mux 客户端配置、服务端 Mux 配置和 TCP Brutal 配置。

## 客户端配置

### Sing-Mux 配置结构

```yaml
proxies:
  - name: "node-name"
    type: trojan  # 或 vless, vmess, ss
    server: example.com
    port: 443
    password: "password"
    smux:
      enabled: true
      protocol: smux
      max-connections: 4
      min-streams: 4
      max-streams: 0
      padding: false
      statistic: false
      only-tcp: false
      brutal-opts:
        enabled: false
        up: "100 Mbps"
        down: "100 Mbps"
```

### 参数详解

#### enabled

```yaml
enabled: true  # 启用多路复用
```

| 值 | 说明 |
|-----|------|
| true | 启用多路复用 |
| false | 不启用（默认） |

#### protocol

```yaml
protocol: smux  # 多路复用协议
```

| 值 | 说明 | 帧开销 | 适用场景 |
|-----|------|--------|----------|
| smux | Smux 协议 | 12 bytes | 简单高效 |
| yamux | Yamux 协议 | 12 bytes | 功能丰富 |
| h2mux | HTTP/2 Mux | HTTP/2 头 | HTTP/2 环境 |

#### max-connections

```yaml
max-connections: 4  # 最大底层连接数
```

| 范围 | 推荐值 | 说明 |
|------|--------|------|
| 1-16 | 4-8 | 控制并发连接数 |

- 每个连接可以承载多个流
- Brutal 模式下需要多连接
- 连接数增加会增加资源消耗

#### min-streams

```yaml
min-streams: 4  # 每连接最小流数
```

- 控制流的集中度
- 建议与 max-connections 相同
- 太小会分散流，降低效率

#### max-streams

```yaml
max-streams: 0  # 每连接最大流数（0=无限制）
```

| 值 | 说明 |
|-----|------|
| 0 | 无限制 |
| >0 | 限制每连接流数 |

- 0 是推荐值
- 非零值用于控制资源

#### padding

```yaml
padding: false  # 随机填充
```

| 值 | 说明 |
|-----|------|
| true | 启用随机填充 |
| false | 不启用（默认） |

- 增加流量隐蔽性
- 增加带宽开销
- 对抗流量分析

#### statistic

```yaml
statistic: false  # 流量统计
```

| 值 | 说明 |
|-----|------|
| true | 启用统计 |
| false | 不启用（默认） |

- 记录流量使用情况
- 用于调试和监控

#### only-tcp

```yaml
only-tcp: false  # 仅处理 TCP
```

| 值 | 说明 |
|-----|------|
| true | UDP 不走 Mux |
| false | TCP/UDP 都走 Mux |

- UDP Mux 可能有问题
- 某些场景需要 true

## TCP Brutal 配置

### brutal-opts 结构

```yaml
brutal-opts:
  enabled: true
  up: "100 Mbps"
  down: "200 Mbps"
```

### enabled

```yaml
enabled: true  # 启用 TCP Brutal
```

- 需要服务端支持
- 激进的多路复用优化
- Linux 系统效果更好

### up / down

```yaml
up: "100 Mbps"    # 上行带宽
down: "200 Mbps"  # 下行带宽
```

带宽格式：

| 格式 | 示例 | 说明 |
|------|------|------|
| Mbps | `100 Mbps` | Megabits/s |
| Gbps | `1 Gbps` | Gigabits/s |
| KB/s | `100 KB/s` | Kilobytes/s |
| MB/s | `100 MB/s` | Megabytes/s |
| 数值 | `10000000` | Bytes/s |

带宽配置建议：
- 准确配置实际带宽
- up = 客户端上行带宽
- down = 客户端下行带宽
- 过高配置可能导致拥塞

## 服务端配置

### Mux 服务端配置

```yaml
listeners:
  - name: trojan-in
    type: trojan
    listen: :443
    users:
      - password: "password1"
      - password: "password2"
    mux:
      padding: false
      brutal:
        enabled: true
        up: "1 Gbps"
        down: "1 Gbps"
```

### 服务端 Mux 参数

| 参数 | 说明 |
|------|------|
| padding | 随机填充 |
| brutal.enabled | 启用 Brutal |
| brutal.up | 服务端上行带宽 |
| brutal.down | 服务端下行带宽 |

## 配置示例

### 基础配置

```yaml
proxies:
  - name: "basic-mux"
    type: trojan
    server: server.com
    port: 443
    password: "pass"
    smux:
      enabled: true
      protocol: smux
```

### 高性能配置

```yaml
proxies:
  - name: "high-performance"
    type: vless
    server: server.com
    port: 443
    uuid: "uuid"
    smux:
      enabled: true
      protocol: h2mux
      max-connections: 8
      min-streams: 8
```

### Brutal 配置

```yaml
proxies:
  - name: "brutal-node"
    type: trojan
    server: server.com
    port: 443
    password: "pass"
    smux:
      enabled: true
      protocol: h2mux
      max-connections: 16
      brutal-opts:
        enabled: true
        up: "200 Mbps"
        down: "500 Mbps"
```

### UDP 单独配置

```yaml
proxies:
  - name: "tcp-only-mux"
    type: vmess
    server: server.com
    port: 443
    uuid: "uuid"
    smux:
      enabled: true
      protocol: smux
      only-tcp: true  # UDP 直连
```

## 参数对照表

| 参数 | 客户端 | 服务端 | 说明 |
|------|--------|--------|------|
| enabled | 支持 | - | 启用 Mux |
| protocol | 支持 | - | 协议类型 |
| max-connections | 支持 | - | 最大连接 |
| min-streams | 支持 | - | 最小流数 |
| max-streams | 支持 | - | 最大流数 |
| padding | 支持 | 支持 | 随机填充 |
| statistic | 支持 | - | 流量统计 |
| only-tcp | 支持 | - | 仅 TCP |
| brutal.enabled | 支持 | 支持 | Brutal |
| brutal.up | 支持 | 支持 | 上行带宽 |
| brutal.down | 支持 | 支持 | 下行带宽 |

## 相关链接

- [[ref/mihomo/mux/overview|Mux 概览]] — 多路复用总览
- [[ref/mihomo/mux/smux|Smux]] — Smux 协议详解
- [[ref/mihomo/mux/yamux|Yamux]] — Yamux 协议详解
- [[ref/mihomo/mux/singmux|Sing-Mux]] — Sing-Mux 协议详解