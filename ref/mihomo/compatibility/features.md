---
title: "功能支持矩阵"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/compatibility
source: "mihomo-Meta/"
tags: [mihomo, features, matrix, support, capability]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/compatibility/overview]]"
  - "[[ref/mihomo/compatibility/prism]]"
  - "[[ref/mihomo/compatibility/protocol-matrix]]"
---

# 功能支持矩阵

**类别**: Mihomo 兼容性 | **模块**: 功能支持状态

## 概述

本页面列出 mihomo 支持的所有功能特性及其状态，帮助用户了解 mihomo 的能力范围。

## 核心功能

### 代理功能

| 功能 | 支持 | 说明 |
|------|------|------|
| TCP 代理 | 是 | 所有协议支持 TCP |
| UDP 代理 | 是 | 大部分协议支持 UDP |
| Fullcone NAT | 是 | 部分 UDP 卯议支持 |
| 多路复用 | 是 | Smux/Yamux/h2mux |
| TCP Brutal | 是 | 高性能多路复用 |

### DNS 功能

| 功能 | 支持 | 说明 |
|------|------|------|
| fake-ip 模式 | 是 | 推荐，防泄露 |
| redir-host 模式 | 是 | 真实解析模式 |
| DNS 缓存 | 是 | 内置缓存 |
| DoH | 是 | DNS over HTTPS |
| DoT | 是 | DNS over TLS |
| DNS 分流 | 是 | nameserver/fallback |
| DNS 规则 | 是 | 独立 DNS 规则 |
| hosts | 是 | 自定义解析 |

### TUN 模式

| 功能 | 支持 | 说明 |
|------|------|------|
| TUN 模式 | 是 | 透明代理 |
| gVisor Stack | 是 | 用户态网络栈 |
| System Stack | 是 | 内核态网络栈 |
| Mixed Stack | 是 | TCP system + UDP gvisor |
| DNS 劫持 | 是 | UDP/TCP DNS 劫持 |
| 自动路由 | 是 | auto-route |
| 自动接口检测 | 是 | auto-detect-interface |
| 进程识别 | 是 | 进程规则 |

### 规则系统

| 功能 | 支持 | 说明 |
|------|------|------|
| 域名规则 | 是 | DOMAIN/SUFFIX/KEYWORD/REGEX |
| IP 规则 | 是 | IP-CIDR/GeoIP |
| GeoSite 规则 | 是 | 基于域名分类 |
| GeoIP 规则 | 是 | 基于国家/地区 |
| 端口规则 | 是 | SRC-PORT/DST-PORT |
| 进程规则 | 是 | PROCESS-NAME |
| 规则集 | 是 | RULE-SET 外部规则 |
| Script 规则 | 是 | 自定义脚本 |

### 代理组

| 功能 | 支持 | 说明 |
|------|------|------|
| select | 是 | 手动选择 |
| url-test | 是 | 自动测速 |
| fallback | 是 | 故障转移 |
| load-balancing | 是 | 负载均衡 |
| relay | 是 | 链式代理 |
| Provider | 是 | 外部节点源 |

### Sniffing

| 功能 | 支持 | 说明 |
|------|------|------|
| HTTP 嗅探 | 是 | 从 HTTP 头提取域名 |
| TLS 嗅探 | 是 | 从 SNI 提取域名 |
| QUIC 嗅探 | 是 | 从 QUIC SNI 提取域名 |
| override-destination | 是 | 替换目标域名 |

## 安全特性

### TLS 特性

| 功能 | 支持 | 说明 |
|------|------|------|
| TLS 1.2 | 是 | 兼容旧服务器 |
| TLS 1.3 | 是 | 推荐，现代 TLS |
| 自定义 SNI | 是 | servername/sni |
| Skip Cert Verify | 是 | 自签名证书 |
| ALPN | 是 | HTTP/2 协商 |
| Client Fingerprint | 是 | 自定义 Client Hello |

### 隐蔽特性

| 功能 | 支持 | 说明 |
|------|------|------|
| Reality | 是 | TLS 隐蔽技术 |
| ECH | 是 | Encrypted Client Hello |
| ShadowTLS | 是 | TLS 混淆 |
| Restls | 是 | 响应模拟 |
| AnyTLS | 是 | TLS 特征隐蔽 |

### 加密算法

| 算法 | 支持 | 说明 |
|------|------|------|
| AES-GCM | 是 | AES-128/256-GCM |
| ChaCha20-Poly1305 | 是 | 推荐移动设备 |
| XChaCha20-Poly1305 | 是 | 扩展 nonce |
| Blake3 | 是 | 高性能哈希 |
| HKDF | 是 | 密钥派生 |

## 高级功能

### 连接管理

| 功能 | 支持 | 说明 |
|------|------|------|
| TCP Concurrent | 是 | TCP 并发优化 |
| Unified Delay | 是 | 统一延迟计算 |
| Keepalive | 是 | 保活机制 |
| Connection Pool | 是 | 连接池 |
| Lazy 测速 | 是 | 懒加载测速 |

### 性能优化

| 功能 | 支持 | 说明 |
|------|------|------|
| TCP Brutal | 是 | 激进多路复用 |
| UDP Over TCP | 是 | UDP 转 TCP |
| Batch 测速 | 是 | 批量测速 |

### 调试功能

| 功能 | 支持 | 说明 |
|------|------|------|
| Debug 日志 | 是 | 详细日志输出 |
| API 接口 | 是 | RESTful API |
| Web UI | 是 | metacubexd 等 |
| 流量统计 | 是 | 上传/下载统计 |

## 平台支持

| 平台 | 支持 | 说明 |
|------|------|------|
| Windows | 是 | WinTUN 驱动 |
| macOS | 是 | utun 驱动 |
| Linux | 是 | /dev/net/tun |
| Android | 是 | VPN API |
| iOS | 是 | Network Extension |
| Docker | 是 | 官方镜像 |

## 数据支持

| 数据 | 支持 | 说明 |
|------|------|------|
| GeoIP.dat | 是 | 旧格式 |
| GeoIP.mmdb | 是 | MaxMind 格式 |
| GeoSite.dat | 是 | 域名分类 |
| ASN.mmdb | 是 | ASN 数据 |
| 自动更新 | 是 | 定期更新 GEO 数据 |

## 不支持功能

| 功能 | 状态 | 说明 |
|------|------|------|
| IPv6 纯净代理 | 部分 | 部分协议支持 |
| SSTP | 不支持 | Windows VPN |
| L2TP/IPSec | 不支持 | 传统 VPN |
| OpenVPN | 不支持 | 传统 VPN |
| IKEv2 | 不支持 | 传统 VPN |

## 版本差异

不同版本功能可能不同:
- 增强版 (Meta): 功能最全
- 核心版: 基础功能
- 兼容版: Clash 原版功能

## 相关链接

- [[overview]] — 兼容性总览
- [[prism]] — Prism 兼容性
- [[protocol-matrix]] — 协议支持矩阵
- [[ref/mihomo/config/overview]] — 配置总览