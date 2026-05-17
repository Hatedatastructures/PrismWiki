---
title: "Keepalive 实现"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/implementation
source: "mihomo-Meta/dialer/"
tags: [mihomo, keepalive, connection, pool, timeout, implementation]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/implementation/overview]]"
  - "[[ref/mihomo/config/advanced.yaml]]"
---

# Keepalive 实现

**类别**: Mihomo 实现机制 | **模块**: 连接保活详解

## 概述

Keepalive（保活）是 mihomo 的连接维护功能，确保长连接稳定，减少重新建立连接的开销。

## 配置

```yaml
# Keepalive 配置
keep-alive-interval: 30    # 保活间隔（秒）
```

## 工作原理

### TCP Keepalive

TCP Keepalive 检测连接状态:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TCP Keepalive 机制                                │
└─────────────────────────────────────────────────────────────────────┘

正常连接状态
    │
    ▼
┌─────────────────────┐
│  定时发送 Keepalive │
│  (每隔 N 秒)        │
│  空 ACK 包          │
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│  等待响应           │
│                     │
│  收到 ACK: 连接正常 │
│  无响应: 连接异常   │
└─────────────────────┘
    │
    ┌─────────────────┴─────────────────┐
    │                                   │
    ▼                                   ▼
┌─────────────┐                   ┌─────────────┐
│  连接正常   │                   │  连接异常   │
│  继续使用   │                   │  关闭连接   │
└─────────────┘                   │  重新建立   │
                                  └─────────────┘
```

### HTTP Keepalive

HTTP 连接的保活:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HTTP Keepalive                                   │
└─────────────────────────────────────────────────────────────────────┘

HTTP/1.1 Keep-Alive:
┌─────────────────────────────────────────────────────────────────────┐
│  Connection: Keep-Alive                                             │
│  Keep-Alive: timeout=60, max=100                                   │
│                                                                      │
│  - 连接可复用                                                       │
│  - 超时 60 秒                                                       │
│  - 最多 100 请求                                                    │
└─────────────────────────────────────────────────────────────────────┘

HTTP/2 Multiplexing:
┌─────────────────────────────────────────────────────────────────────┐
│  - 单连接多请求                                                     │
│  - 无需额外 Keepalive                                               │
│  - 内置流管理                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

## 配置详解

### keep-alive-interval

```yaml
keep-alive-interval: 30    # 保活探测间隔
```

| 值 | 说明 |
|-----|------|
| 0 | 禁用 Keepalive |
| 30 | 默认，30 秒探测一次 |
| 60 | 较长间隔，减少开销 |
| 15 | 较短间隔，更快检测 |

### 配合其他配置

```yaml
# 连接管理配置
keep-alive-interval: 30    # 保活间隔

# 连接池
# (内部自动管理)

# 超时设置
# (协议级别配置)
```

## 连接池管理

### 连接池原理

```
┌─────────────────────────────────────────────────────────────────────┐
│                        连接池管理                                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────┐
│     连接池          │
│                     │
│  ┌─────┐ ┌─────┐   │
│  │ C1  │ │ C2  │   │  活跃连接
│  └─────┘ └─────┘   │
│                     │
│  ┌─────┐ ┌─────┐   │
│  │ C3  │ │ C4  │   │  备用连接
│  └─────┘ └─────┘   │
│                     │
└─────────────────────┘

新请求到达
    │
    ▼
┌─────────────────────┐
│  从池中获取连接     │
│                     │
│  有可用: 直接使用   │
│  无可用: 新建连接   │
└─────────────────────┘

请求完成
    │
    ▼
┌─────────────────────┐
│  连接归还池中       │
│                     │
│  连接有效: 放回池   │
│  连接无效: 关闭     │
└─────────────────────┘
```

### 连接池参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| 最大连接数 | 每代理最大连接 | 内部控制 |
| 最大空闲数 | 每代理最大空闲连接 | 内部控制 |
| 空闲超时 | 空闲连接超时时间 | keep-alive-interval |

## Keepalive 与代理协议

### Trojan Keepalive

```yaml
proxies:
  - name: "trojan-keepalive"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    # TLS 连接的 Keepalive 由底层 TCP 处理
```

Trojan 通过 TLS 保活:
- TLS 连接继承 TCP Keepalive
- 加密数据流持续

### VLESS Keepalive

```yaml
proxies:
  - name: "vless-keepalive"
    type: vless
    server: server.com
    port: 443
    uuid: "uuid"
    # 连接保活由 mihomo 管理
```

### Hysteria2 Keepalive

```yaml
proxies:
  - name: "hysteria2-keepalive"
    type: hysteria2
    server: server.com
    port: 443
    password: "password"
    # QUIC 内置保活机制
```

QUIC 内置保活:
- QUIC 有自己的保活机制
- 不依赖 TCP Keepalive
- 更灵活的保活策略

### Smux Keepalive

```yaml
proxies:
  - name: "smux-keepalive"
    type: trojan
    server: server.com
    port: 443
    password: "password"
    smux:
      enabled: true
      # Smux 有内置保活
```

Smux 保活:
- 协议级别心跳
- 保持底层连接活跃

## Keepalive 优势

### 减少连接开销

```
┌─────────────────────────────────────┐
│       连接开销对比                    │
├─────────────────────────────────────┤
│                                      │
│  无 Keepalive:                       │
│  每请求: TCP握手 + TLS握手           │
│  开销: ~100ms + ~200ms               │
│                                      │
│  有 Keepalive:                       │
│  复用连接                            │
│  开销: ~0ms                          │
│                                      │
│  优化效果: 显著减少延迟              │
│                                      │
└─────────────────────────────────────┘
```

### 提高稳定性

```
┌─────────────────────────────────────┐
│       稳定性提升                     │
├─────────────────────────────────────┤
│                                      │
│  Keepalive 功能:                     │
│  - 检测连接状态                      │
│  - 及时发现断连                      │
│  - 自动清理无效连接                  │
│                                      │
│  结果:                               │
│  - 连接更稳定                        │
│  - 错误更少                          │
│  - 体验更好                          │
│                                      │
└─────────────────────────────────────┘
```

## Keepalive 问题排查

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 连接断开频繁 | Keepalive 间隔太长 | 减小 interval |
| 保活不生效 | 协议不支持 | 检查协议 |
| 资源泄漏 | 连接未正确清理 | 检查版本 |

### 调试方法

```bash
# 查看连接状态
netstat -an | grep ESTABLISHED

# 查看保活设置
# Linux
cat /proc/sys/net/ipv4/tcp_keepalive_time
cat /proc/sys/net/ipv4/tcp_keepalive_intvl
cat /proc/sys/net/ipv4/tcp_keepalive_probes

# 查看 mihomo 日志
# 启用 debug 日志观察连接状态
```

## 配置建议

| 场景 | 推荐 interval | 原因 |
|------|---------------|------|
| 稳定网络 | 60 | 减少探测开销 |
| 不稳定网络 | 30 | 更快检测断连 |
| 移动网络 | 15 | 网络变化频繁 |
| 内网 | 120 | 网络稳定 |

## 源码位置

- Keepalive 实现: `dialer/`
- 连接池管理: 内部实现

## 相关链接

- [[overview]] — 实现总览
- [[tcp-concurrent]] — TCP 并发详解
- [[ref/mihomo/config/advanced.yaml]] — 高级配置模板