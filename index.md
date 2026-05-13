# Prism 知识库

> Prism 高性能代理引擎的项目文档 — 覆盖模块设计、实现细节、调试排障、客户端对接、Bug 记录。
> 最后更新：2026-05-13

---

## 核心模块

- [[agent]] — 前端监听、会话管理、负载均衡、协议分发
- [[channel]] — 连接池、Happy Eyeballs、传输层抽象
- [[crypto]] — AEAD、Blake3、X25519、HKDF
- [[memory]] — PMR 内存池、三级池化、帧分配器
- [[fault]] — 错误码枚举、双轨错误处理
- [[exception]] — 异常层次结构
- [[trace]] — spdlog 异步日志
- [[transformer]] — glaze JSON 序列化
- [[loader]] — 配置加载、账户目录构建
- [[outbound]] — 出站代理接口、直连实现

## 协议与管道

- [[protocol]] — 协议编解码层
- [[protocol/common]] — 地址解析、协议表单、UDP 中继公共组件
- [[protocol/tls]] — TLS ClientHello 特征分析、feature_bitmap
- [[pipeline]] — 协议处理管道、原语操作
- [[pipeline/processors]] — 协议处理器详解（5 个处理器）
- [[recognition]] — 协议智能识别（Probe + Identify）

## 协议实现

- [[protocol/http]] — HTTP/1.1 代理协议 + Prism 实现
- [[protocol/socks5]] — SOCKS5 协议 + Prism 实现
- [[protocol/trojan]] — Trojan 协议
- [[protocol/trojan-gfw]] — Trojan-GFW 原始实现
- [[protocol/vless]] — VLESS 协议
- [[protocol/shadowsocks]] — Shadowsocks 2022
- [[protocol/proxy-protocols]] — 代理协议总览与选型

## 传输与伪装

- [[multiplex]] — 多路复用（smux/yamux）
- [[stealth]] — TLS 伪装模块
- [[resolve]] — DNS 解析、路由、缓存

## 伪装方案

- [[stealth/reality]] — Reality TLS 伪装
- [[stealth/shadowtls]] — ShadowTLS v2/v3
- [[stealth/restls]] — Restls
- [[stealth/anytls]] — AnyTLS
- [[stealth/trusttunnel]] — TrustTunnel
- [[stealth/ech]] — ECH 加密 Client Hello
- [[stealth/native]] — 原生 TLS（无伪装 fallback）
- [[stealth/pki-certificates]] — PKI 证书体系
- [[stealth/proxy-detection]] — 代理检测技术

## 多路复用协议

- [[multiplex/smux]] — smux 协议规范
- [[multiplex/yamux]] — yamux 协议规范

## 模块详解

- [[channel/transport]] — 传输层接口分层
- [[crypto/aead]] — AEAD 加密实现
- [[resolve/dns]] — DNS 解析器管道
- [[memory/pool]] — 内存池三级架构

## Agent 子文档

- [[agent/architecture]] — 分层架构设计
- [[agent/configuration]] — 配置详解
- [[agent/deployment]] — 部署指南
- [[agent/testing]] — 测试体系
- [[agent/troubleshooting]] — 故障排查

## 客户端对接

- [[client/mihomo-meta]] — mihomo 客户端概述
- [[client/mihomo-clash-config]] — mihomo 配置详解
- [[client/mihomo-proxy-groups]] — 代理组配置
- [[client/mihomo-rules]] — 规则系统
- [[client/mihomo-dns]] — DNS 处理机制
- [[client/tun]] — TUN 模式

## 开发笔记

- [[dev/modules]] — Prism 全局模块结构
- [[dev/cpp23-coroutines]] — C++23 协程
- [[dev/pmr-memory-pool]] — PMR 内存池
- [[dev/tcp]] — TCP 协议基础
- [[dev/tls]] — TLS 协议基础
- [[dev/udp]] — UDP 协议基础
- [[dev/gfw]] — GFW 封锁原理
- [[dev/configuration]] — 完整配置文件参考
- [[dev/testing]] — 测试体系
- [[dev/stress]] — 压力测试

## 性能报告

- [[performance/report]] — 基准测试报告
- [[performance/benchmark]] — 基准测试详解（12 个 bench）

## bugs/ — Bug 记录

> 暂无记录。
