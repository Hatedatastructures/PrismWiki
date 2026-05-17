---
title: "参考资料概览"
category: "ref"
type: ref
layer: ref
module: ref
source: ""
tags: [参考, 知识库, 概览]
created: 2026-05-17
updated: 2026-05-17
---

# 参考资料概览

**类别**: 参考资料

## 概述

参考资料层（ref）是 PrismWiki 知识库的第三层，提供协议规范、加密原理、网络基础等技术参考资料。该层面向需要深入了解技术细节的读者，包含 RFC 规范解析、算法原理说明、协议帧格式等中等层级内容。

### 层级定位

PrismWiki 采用三层知识架构：

```
┌────────────────────────────────────────────────────────────┐
│  Layer 1: docs/  — 应用层文档                               │
│  部署指南、配置说明、使用手册                                │
│  目标读者：运维人员、使用者                                  │
├────────────────────────────────────────────────────────────┤
│  Layer 2: core/  — 实现层文档                              │
│  架构设计、模块实现、代码解析                                │
│  目标读者：开发者、维护者                                    │
├────────────────────────────────────────────────────────────┤
│  Layer 3: ref/  — 参考资料层 ← 本层                         │
│  协议规范、加密原理、网络基础                                │
│  目标读者：需要深入了解技术原理的读者                        │
└────────────────────────────────────────────────────────────┘
```

参考资料层补充实现层文档中引用的技术背景知识，为读者提供完整的知识体系。

### 内容分类

参考资料层按技术领域分为以下子目录：

| 目录 | 内容范围 | 关键主题 |
|------|----------|----------|
| [[ref/crypto|crypto]] | 密码学原理 | AEAD、密钥交换、HKDF、TLS 加密 |
| [[ref/protocol|protocol]] | 协议规范 | TLS 握手、SOCKS5、HTTP 代理、TCP/UDP/QUIC |
| [[ref/network|network]] | 网络基础 | Happy Eyeballs、连接池、DNS 解析、GFW |
| [[ref/programming|programming]] | 编程技术 | Boost.Asio、C++23 协程、constexpr |
| [[ref/memory|memory]] | 内存管理 | PMR、Arena、零拷贝 |
| [[ref/anti-censorship|anti-censorship]] | 反审查技术 | TLS 指纹、DPI、流量分析 |
| [[ref/logging|logging]] | 日志系统 | spdlog、日志框架 |

### 文档特点

参考资料层文档具有以下特点：

**中等层级**：介于 RFC 规范和实现代码之间，提供原理说明和关键格式解析，不包含完整实现细节。

**协议帧格式**：详细解析协议数据帧的结构、字段含义、序列化方式。

**原理说明**：解释算法设计原理、安全考量、性能特性。

**wikilink 引用**：使用 Obsidian wikilink 语法链接相关文档，形成知识网络。

### 使用方式

参考资料层文档的使用方式：

1. **查阅 RFC 解析**：当需要理解协议规范但不想阅读完整 RFC 时
2. **理解算法原理**：当需要了解加密算法、密钥派生等原理时
3. **学习网络基础**：当需要了解 TCP、UDP、DNS 等网络原理时
4. **深入实现背景**：阅读实现层文档时，通过 wikilink 跳转了解背景知识

## 子目录概览

### [[ref/crypto|密码学原理]]

密码学原理目录提供加密算法的原理说明：

| 文件 | 内容 | 参见 |
|------|------|------|
| [[ref/crypto/overview|overview]] | 密码学概览 | 本目录索引 |
| [[ref/crypto/aead-basics|aead-basics]] | AEAD 认证加密原理 | [[crypto/aead]] |
| [[ref/crypto/key-exchange|key-exchange]] | 密钥交换原理 (X25519) | [[crypto/x25519]] |
| [[ref/crypto/hkdf-theory|hkdf-theory]] | HKDF 密钥派生理论 | [[crypto/hkdf]] |
| [[ref/crypto/tls-crypto|tls-crypto]] | TLS 加密原理 | [[ref/protocol/tls-handshake]] |

### [[ref/protocol|协议规范]]

协议规范目录提供网络协议的规范解析：

| 文件 | 内容 | 参见 |
|------|------|------|
| [[ref/protocol/overview|overview]] | 协议规范概览 | 本目录索引 |
| [[ref/protocol/tls-handshake|tls-handshake]] | TLS 握手流程 | [[ref/crypto/tls-crypto]] |
| [[ref/protocol/socks5-spec|socks5-spec]] | SOCKS5 RFC 1928 | [[protocol/socks5]] |
| [[ref/protocol/http-proxy-spec|http-proxy-spec]] | HTTP 代理规范 | [[protocol/http]] |
| [[ref/protocol/tcp-basics|tcp-basics]] | TCP 基础 | [[ref/network/tcp]] |
| [[ref/protocol/udp-basics|udp-basics]] | UDP 基础 | [[ref/network/udp]] |
| [[ref/protocol/quic-basics|quic-basics]] | QUIC 基础 | [[core/multiplex/overview|multiplex]] |

### [[ref/network|网络基础]]

网络基础目录提供网络技术的基础原理：

| 文件 | 内容 | 参见 |
|------|------|------|
| [[ref/network/overview|overview]] | 网络基础概览 | 本目录索引 |
| [[ref/network/happy-eyeballs|happy-eyeballs]] | RFC 6555 Happy Eyeballs | [[channel/eyeball]] |
| [[ref/network/connection-pool|connection-pool]] | 连接池原理 | [[channel/connection/pool]] |
| [[ref/network/dns-resolution|dns-resolution]] | DNS 解析原理 | [[resolve/dns]] |
| [[ref/network/gfw|gfw]] | GFW 原理 | [[core/stealth/overview|stealth]] |
| [[ref/network/proxy-detection|proxy-detection]] | 代理检测原理 | [[core/recognition/overview|recognition]] |

## 参考资料

- [[index|知识库索引]] — 知识库总览
- [[core/crypto/overview|Crypto 模块]] — 加密算法实现
- [[core/protocol/analysis|协议分析]] — 协议实现层
- [[core/stealth/overview|伪装层]] — TLS 伪装方案