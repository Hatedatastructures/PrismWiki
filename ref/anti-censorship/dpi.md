---
title: "深度包检测 (DPI)"
category: "anti-censorship"
type: ref
tags: [流量对抗, dpi, 检测]
created: 2026-05-15
updated: 2026-05-15
---

# 深度包检测 (DPI)

**类别**: 流量对抗

## 概述

深度包检测（Deep Packet Inspection）是网络审查的核心技术，通过分析数据包的载荷内容来识别和封锁特定协议或服务。

## 原理

### 检测方式

| 方式 | 说明 | 特点 |
|------|------|------|
| 旁路观测 | 镜像流量分析 | 不影响性能，但有延迟 |
| 在线检测 | 实时分析并阻断 | 低延迟，但影响性能 |

### 检测技术

1. **协议解析**: 解析应用层协议（HTTP、TLS 等）
2. **特征匹配**: 匹配已知协议特征（如 TLS ClientHello 格式）
3. **行为分析**: 分析流量模式（包大小、时序、方向）
4. **机器学习**: 使用 ML 模型识别未知协议

### TLS 特征

DPI 可以检测的 TLS 特征：
- ClientHello 格式和扩展
- 证书信息
- 密码套件列表
- SNI 域名

## 在 Prism 中的应用

Prism 通过多种伪装方案对抗 DPI：
- **Reality**: 伪装成真实网站的 TLS 流量
- **ShadowTLS**: 使用标准 TLS 外层
- **ECH**: 加密 SNI，防止域名泄露

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| stealth_scheme | 伪装方案基类 | [[stealth/scheme|scheme]] |
| sniff | Tier 0 快速检测 | [[stealth/scheme|scheme]] |

## 参考资料

- [Great Firewall of China](https://en.wikipedia.org/wiki/Great_Firewall)

## 相关知识

- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — JA3/JA4 指纹
- [[ref/anti-censorship/traffic-analysis|流量分析]] — 流量分析技术
- [[ref/anti-censorship/probing|主动探测]] — 主动探测防御
