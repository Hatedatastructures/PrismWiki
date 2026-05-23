---
title: "网络基础概览"
category: "network"
type: ref
layer: ref
module: ref
source: []
tags: [网络, 概览, dns, 连接池, happy-eyeballs, gfw]
created: 2026-05-17
updated: 2026-05-17
---

# 网络基础概览

**类别**: 网络

## 概述

网络基础目录提供网络技术的原理说明，涵盖连接管理、DNS 解析、反审查技术等内容。这些原理是理解代理服务器行为和网络环境的基础。

### 内容分类

网络基础涵盖以下类别：

| 类别 | 内容 | 关键主题 |
|------|------|----------|
| 连接管理 | Happy Eyeballs、连接池 | IPv4/IPv6 并行、连接复用 |
| DNS 解析 | DNS 原理、DNS-over-X | 传统 DNS、加密 DNS |
| 反审查 | GFW 原理、代理检测 | DPI、流量分析 |

### 在 Prism 中的应用

Prism 项目中网络原理的应用：

| 技术 | 应用场景 | 实现模块 |
|------|----------|----------|
| Happy Eyeballs | 双栈连接 | [[core/connect/dial|eyeball]] |
| 连接池 | 连接复用 | [[core/connect/pool/pool|connection-pool]] |
| DNS 解析 | DNS 查询 | [[core/resolve/dns|dns]] |
| GFW 知识 | 伪装设计 | [[core/stealth/overview|stealth]] |

## 连接管理

### Happy Eyeballs

Happy Eyeballs 是双栈连接算法：

```
Happy Eyeballs (RFC 6555)：

问题：
- IPv4/IPv6 双栈网络
- IPv6 连接可能失败或慢
- 传统串行尝试浪费时间

解决方案：
- 并行发起 IPv4 和 IPv6 连接
- IPv6 略优先（延迟 50-250ms）
- 使用第一个成功的连接

流程：
┌─────────────────────────────────────────────┐
│ DNS 解析 → 获得 IPv4 + IPv6 地址            │
│                                             │
│ IPv6 地址 → 异步连接尝试                    │
│                                             │
│ [等待 50-250ms]                             │
│                                             │
│ IPv4 地址 → 异步连接尝试                    │
│                                             │
│ 使用第一个成功的连接                        │
│ 取消其他尝试                                │
└─────────────────────────────────────────────┘
```

详见 [[ref/network/happy-eyeballs|Happy Eyeballs]]。

### 连接池

连接池复用 TCP 连接：

```
连接池原理：

目的：
- 减少 TCP 握手开销
- 复用已建立的连接
- 提升吞吐量

管理策略：
- 最大连接数限制
- 最小保持连接数
- 连接空闲超时
- 连接健康检查

池化协议：
- HTTP/1.1 Keep-Alive
- HTTP/2 多路复用
- 自定义多路复用（smux/yamux）
```

详见 [[ref/network/connection-pool|连接池原理]]。

## DNS 解析

### DNS 基础

DNS 解析的基本流程：

```
DNS 解析流程：

1. 客户端查询本地 resolver
   query: www.example.com A

2. Resolver 检查缓存
   - 有缓存 → 直接返回
   - 无缓存 → 递归查询

3. 递归查询流程：
   Resolver → Root DNS → .com NS
   Resolver → .com NS → example.com NS
   Resolver → example.com NS → IP 地址

4. Resolver 缓存结果
   TTL: Time To Live

5. 返回给客户端

记录类型：
- A: IPv4 地址
- AAAA: IPv6 地址
- CNAME: 别名
- NS: 名称服务器
- MX: 邮件交换
```

详见 [[ref/network/dns-resolution|DNS 解析原理]]。

### DNS-over-X

加密 DNS 的多种方式：

| 方式 | 传输 | 特点 |
|------|------|------|
| DNS-over-UDP | UDP 53 | 传统方式，快速但无隐私 |
| DNS-over-TCP | TCP 53 | 大响应，可靠但无隐私 |
| DNS-over-TLS | TLS 853 | 加密，DoT 标准 |
| DNS-over-HTTPS | HTTPS 443 | 加密，穿透防火墙 |

详见 [[ref/protocol/dns-over-tls]], [[ref/protocol/dns-over-https]]。

## 反审查技术

### GFW 原理

防火墙的工作原理：

```
GFW 检测技术：

1. IP 封锁
   - 黑名单 IP 无法访问
   - 路由黑洞

2. DNS 污染
   - 返回错误 IP
   - 拦截 DNS 查询

3. DPI (Deep Packet Inspection)
   - 检测 TLS ClientHello 特征
   - 检测协议指纹
   - SNI 封锁

4. 流量分析
   - 检测加密流量特征
   - 统计分析流量模式

5. 主动探测
   - 检测可疑流量
   - 主动连接验证
```

详见 [[ref/network/gfw|GFW 原理]]。

### 代理检测

代理服务器的检测方法：

```
代理检测方法：

1. TLS 指纹
   - ClientHello 特征识别
   - JA3/JA3S 签名
   - 非标准指纹被标记

2. 行为特征
   - 连接模式异常
   - 流量特征异常
   - 时间模式异常

3. 主动探测
   - 模拟客户端行为
   - 验证服务响应
   - 区分代理和真实服务

4. IP 段分析
   - 云服务 IP 封锁
   - VPS 提供商限制
```

详见 [[ref/network/proxy-detection|代理检测原理]]。

## 子目录索引

| 文件 | 内容 | 链接 |
|------|------|------|
| Happy Eyeballs | RFC 6555 双栈连接 | [[ref/network/happy-eyeballs]] |
| 连接池原理 | 连接复用机制 | [[ref/network/connection-pool]] |
| DNS 解析原理 | DNS 查询机制 | [[ref/network/dns-resolution]] |
| GFW 原理 | 防火墙技术 | [[ref/network/gfw]] |
| 代理检测原理 | 代理识别方法 | [[ref/network/proxy-detection]] |
| TCP 协议 | TCP 详细原理 | [[ref/network/tcp]] |
| UDP 协议 | UDP 详细原理 | [[ref/network/udp]] |
| NAT 穿透 | NAT 原理与穿透 | [[ref/network/nat-traversal]] |

## 参见

- [[ref/protocol/overview|协议规范概览]] — 协议层索引
- [[core/transport/overview|Channel 模块]] — 传输实现
- [[core/resolve/dns|DNS 模块]] — DNS 实现
- [[core/stealth/overview|Stealth 模块]] — 伪装方案
- [[ref/overview|参考资料概览]] — 参考资料层索引