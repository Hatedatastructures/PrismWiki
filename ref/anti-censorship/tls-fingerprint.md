---
title: "TLS 指纹 (JA3/JA4)"
category: "anti-censorship"
type: ref
tags: [流量对抗, tls, 指纹, ja3, ja4]
created: 2026-05-15
updated: 2026-05-15
---

# TLS 指纹 (JA3/JA4)

**类别**: 流量对抗

## 概述

TLS 指纹通过分析 ClientHello 的特征（版本、密码套件、扩展等）生成唯一标识，用于识别客户端软件或检测代理流量。

## 原理

### JA3 指纹

JA3 通过以下字段生成 MD5 哈希：
```
JA3 = MD5(SSLVersion,Ciphers,Extensions,EllipticCurves,EllipticCurvePointFormats)
```

### JA4 指纹

JA4 是 JA3 的改进版本，提供更细粒度的识别：
```
JA4 = TLSVersion_Ciphers_Extensions_SNI_ALPN
```

### 检测方式

DPI 使用 TLS 指纹：
- 识别代理软件（如 V2Ray、Shadowsocks）
- 检测非标准 TLS 实现
- 封锁特定客户端

## 在 Prism 中的应用

Prism 通过伪装方案对抗 TLS 指纹检测：
- **Reality**: 使用真实网站的 TLS 指纹
- **ShadowTLS**: 使用标准 TLS 实现
- **特征位图**: 分析 ClientHello 特征

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| client_hello_features | 特征提取 | [[protocol/tls/types|types]] |
| feature_bitmap | 特征位图 | [[protocol/tls/feature_bitmap|feature_bitmap]] |
| analyzer | 特征分析 | [[recognition/probe/analyzer|analyzer]] |

## 参考资料

- [JA3 - A Method for Profiling SSL/TLS Clients](https://github.com/salesforce/ja3)
- [JA4+ TLS Client Fingerprinting](https://github.com/FoxIO-LLC/ja4)

## 相关知识

- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 结构
