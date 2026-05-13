---
title: "代理检测技术"
created: 2026-05-13
updated: 2026-05-13
type: dev
tags: [detection, gfw, fingerprinting, analysis]
related: ["[[dev/gfw]]", "[[stealth]]", "[[stealth/reality]]", "[[stealth/pki-certificates]]"]
---

> 相关：[[dev/gfw]] | [[stealth]] | [[stealth/reality]] | [[stealth/pki-certificates]] | [[protocol/proxy-protocols]] | [[dev/tls]] | [[agent]]

# 代理检测技术

## 1. 概述

审查系统（如中国的 GFW）使用多种技术检测和封锁代理流量。理解这些检测方法是设计隐蔽代理协议的前提。检测技术大致分为被动分析和主动探测两大类。

## 2. 被动流量分析

被动分析不注入任何流量，仅通过观察现有连接的特征进行判断。

### 2.1 流量统计分析

- **包大小分布**：正常 HTTPS 浏览有典型的包大小模式（大包下载 + 小包 ACK），代理流量可能呈现不同的分布
- **时间间隔**：交互式 SSH/远程桌面通过代理会产生特定的时间模式
- **流量突发性**：视频流有明显的突发模式，而加密代理隧道可能更均匀
- **上下行比**：正常浏览上行少下行多，某些代理协议上下行比例异常

### 2.2 包长度序列分析

将连接中每个包的长度组成序列，使用机器学习分类器（如 CUMUL、DF、k-FP）判断是否为代理流量。研究表明，即使流量完全加密，包长度序列仍可泄露应用类型信息。

### 2.3 网站指纹攻击

针对定向封锁场景，通过分析流量模式推断用户访问的具体网站。每个网站有独特的"指纹"——页面加载时产生的请求序列和大小组合。

## 3. 主动探测

主动探测需要审查系统向疑似代理服务器发起连接。

### 3.1 回连探测（Active Probing）

这是 GFW 最有效的检测手段之一：

1. 在正常流量中发现可疑 TLS ClientHello（如使用了非标准的扩展组合）
2. 用相同的 SNI 和目标 IP 向该服务器发起新连接
3. 如果服务器响应与合法网站不一致，则判定为代理

**Shadowsocks 被封事件**：早期 SS 使用固定的加密模式，GFW 向疑似 SS 服务器发送随机数据，如果返回符合 SS 加密格式的响应，即判定为代理。

### 3.2 TLS 握手分析

GFW 在 TLS 握手阶段进行深度检查：

- **证书链验证**：连接后检查证书是否由合法 CA 签发
- **ALPN 协商**：正常的 HTTPS 会协商 `h2` 或 `http/1.1`
- **扩展字段顺序**：不同的 TLS 实现有不同的扩展顺序，异常组合会被标记
- **Supported Groups**：检查客户端支持的密码学曲线是否符合常见浏览器

## 4. TLS 指纹识别

### 4.1 JA3 / JA3S

JA3 从 TLS ClientHello 提取指纹：

```
JA3 = TLSVersion,Ciphers,Extensions,EllipticCurves,EllipticCurvePointFormats
```

每个值用逗号连接，取 MD5 哈希。Chrome、Firefox、Safari 等都有固定的 JA3 值。使用非标准 TLS 库（如 Go 的 `crypto/tls`）产生的 JA3 值与浏览器不同，容易被识别。

### 4.2 JA4 / JA4+

JA4 是 JA3 的改进版本，解决了 JA3 中的歧义和不稳定问题：

- 使用更结构化的提取方式
- 支持 TLS 1.3 和 QUIC
- 提供更细粒度的分类（如 JA4S 用于 ServerHello）

### 4.3 JARM

JARM 通过向目标服务器发送 10 个特制的 TLS ClientHello 并观察 ServerHello 的差异来生成服务器指纹。可以区分不同类型的 TLS 服务器——如 Nginx、Apache、Caddy，以及 Reality 伪装的真实站点。

### 4.4 TLS 1.3 特有指纹

TLS 1.3 的 ClientHello 结构更加固定，但也带来了新的指纹维度：

- **Key Share 扩展**：支持的曲线和公钥格式
- **PSK 模式**：是否支持 PSK 以及支持的模式
- **压缩扩展**：证书压缩算法的选择
- **ECH 扩展**：Encrypted Client Hello 的使用模式

## 5. 协议指纹

### 5.1 Shadowsocks

- 固定的加密头部格式
- 特定的 IV/Nonce 生成模式
- 包长度分布特征

### 5.2 Trojan

- TLS 握手后第一个数据包包含密码哈希
- 如果哈希不匹配，Trojan 会转发流量到合法网站
- 这种"降级"行为本身可能被探测

### 5.3 VLESS

- 依赖 TLS 层提供加密
- 协议头部简洁，减少指纹特征
- 但仍需面对 TLS 层面的指纹问题

## 6. DNS 层面的检测

- **SNI 分析**：TLS ClientHello 中的 SNI 字段明文传输，GFW 可以精确匹配域名关键词
- **DNS 查询模式**：频繁查询新域名、使用 DoH/DoT 等加密 DNS 本身可能触发监控
- **DNS 污染**：直接返回错误的 IP 地址，阻止用户连接代理服务器

## 7. Prism 的反检测策略

Prism 采用多层防御策略：

| 检测方式 | 防御手段 | 对应模块 |
|----------|----------|----------|
| TLS 指纹 | Reality 协议复用真实 TLS 握手 | [[stealth/reality]] |
| 证书检测 | 使用真实网站证书，通过 CT 验证 | [[stealth/pki-certificates]] |
| 主动探测 | Reality 的探测者看到的是真实站点 | [[stealth/reality]] |
| 流量分析 | 多路复用 + 流量填充 | [[multiplex]] |
| 协议指纹 | 简洁协议设计，无固定模式 | [[protocol/proxy-protocols]] |

## 8. 检测与反检测的演进

GFW 和代理工具之间是持续的军备竞赛。关键趋势：

- **机器学习**：GFW 越来越多地使用 ML 分类器，需要更精细的流量伪装
- **QUIC/HTTP3**：新的传输协议带来新的指纹维度
- **ECH (Encrypted Client Hello)**：有望加密 SNI，但目前部署有限
- **域前置（Domain Fronting）**：CDN 层面的伪装，但主流 CDN 已禁止

Prism 的设计原则是：**协议的每一层都必须与合法流量无差别**——从 TLS 握手到应用层数据，都不能留下可区分的特征。
