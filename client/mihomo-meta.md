---
title: mihomo-Meta
created: 2026-05-12
updated: 2026-05-12
type: entity
tags: [proxy-client, go, clash, configuration]
sources:
  - I:/code/Prism/logs/mihomo-Meta/README.md
confidence: high
---

# mihomo-Meta

mihomo-Meta 是一个基于 Go 语言开发的代理客户端，支持多种代理协议，是 Clash/Mihomo 的增强分支。

## 概述

mihomo-Meta（原名 Clash Meta）是一个功能强大的代理客户端，支持 VMess、VLESS、Shadowsocks、Trojan、Snell、TUIC、Hysteria 等多种协议。它基于 Dreamacro/clash 项目开发，增加了许多新特性。

## 核心特性

### 代理协议支持

- **VMess** — V2Ray 原生协议
- **VLESS** — 轻量级协议
- **Shadowsocks** — 经典代理协议
- **Trojan** — TLS 伪装协议
- **Snell** — Shadowsocks 增强版
- **TUIC** — 基于 QUIC 的协议
- **Hysteria** — 基于 QUIC 的高速协议

### 功能特性

- 本地 HTTP/HTTPS/SOCKS 服务器，支持认证
- 内置 DNS 服务器，支持 DoH/DoT 上游和 fake IP
- 基于域名、GEOIP、IPCIDR 或进程的规则分流
- 远程分组，支持自动故障转移、负载均衡或基于延迟的自动选择
- 远程 providers，允许远程获取节点列表
- Netfilter TCP 重定向，支持 iptables 部署
- 完整的 HTTP RESTful API 控制器

### Web 面板

- **metacubexd** — 官方 Web 面板
- 地址：https://github.com/MetaCubeX/metacubexd

## 技术栈

- **语言**：Go 1.20+
- **构建工具**：Go modules
- **许可证**：GPL-3.0

## mihomo 的核心功能列表

### 规则分流

mihomo 支持多种规则类型，实现精细化的流量分流：

- **DOMAIN** — 域名精确匹配
- **DOMAIN-SUFFIX** — 域名后缀匹配
- **DOMAIN-KEYWORD** — 域名关键词匹配
- **GEOIP** — 基于 GeoIP 数据库的 IP 匹配
- **IPCIDR** — IP 地址段匹配
- **SRC-IPCIDR** — 源 IP 地址段匹配
- **PROCESS-NAME** — 进程名称匹配（需要 TUN 或系统代理）
- **RULE-SET** — 规则集，支持远程更新
- **MATCH** — 兜底规则

规则按顺序匹配，第一个命中的规则决定流量走向。

### DNS

mihomo 内置完整的 DNS 服务器：

- **DNS 模式** — normal（真实 IP）和 fake-ip（虚拟 IP）两种模式
- **上游协议** — UDP、TCP、DoH（DNS over HTTPS）、DoT（DNS over TLS）
- **DNS 缓存** — 支持 DNS 缓存，减少上游查询
- **hosts 文件** — 支持自定义 DNS 解析
- **DNS 分流** — 不同域名使用不同的 DNS 上游
- **IPv6** — 支持 IPv6 DNS 解析

详见 [[client/mihomo-dns]]。

### TUN 模式

TUN 模式通过虚拟网卡接管系统全局流量：

- 支持 Android、iOS、macOS、Windows、Linux
- 自动配置路由和 DNS
- 支持 IPv4 和 IPv6
- 可配合 tun-stack 使用（gvisor / system）

详见 [[client/tun]]。

### 代理组

代理组是 mihomo 的核心概念，用于组织和管理代理节点：

- **select** — 手动选择
- **url-test** — 自动测速选择延迟最低的节点
- **fallback** — 故障转移，按顺序尝试节点
- **load-balance** — 负载均衡（支持 round-robin、consistent-hashing 等策略）
- **relay** — 链式代理，流量依次经过多个节点

代理组可以嵌套，实现复杂的路由策略。

### API

mihomo 提供完整的 RESTful API：

- `GET /configs` — 获取当前配置
- `PATCH /configs` — 更新配置
- `GET /proxies` — 获取代理组和节点列表
- `PUT /proxies/:name` — 切换代理组选中的节点
- `GET /rules` — 获取规则列表
- `GET /connections` — 获取活跃连接列表
- `DELETE /connections` — 关闭所有连接
- `GET /providers/proxies` — 获取代理 provider 列表

API 默认监听在 mixed-port + 1 的端口上，可通过 `external-controller` 配置修改。

## mihomo 与原版 Clash 的区别

| 特性 | mihomo (Clash Meta) | 原版 Clash (Dreamacro) |
|------|---------------------|----------------------|
| 协议支持 | 更多（VLESS、TUIC、Hysteria、SS2022） | 较少（VMess、Trojan、SS） |
| 规则类型 | 更丰富（RULE-SET、GEOSITE 等） | 基础规则类型 |
| TUN 模式 | 完整支持，多平台 | 基础支持 |
| 代理组 | 支持 relay 链式代理 | 不支持 relay |
| TLS 指纹 | 支持 uTLS/client-fingerprint | 不支持 |
| Reality | 支持 | 不支持 |
| Script | 支持 JavaScript 脚本规则 | 不支持 |
| 维护状态 | 持续活跃 | 已停止维护 |
| 社区生态 | 活跃，metacubexd 面板 | 生态萎缩 |

mihomo 是原版 Clash 的超集，所有原版 Clash 的配置在 mihomo 中都能正常工作。

## 支持的代理协议完整列表

mihomo 支持的代理协议：

| 协议 | 传输层 | 加密 | 特点 |
|------|--------|------|------|
| Shadowsocks | TCP/UDP | AEAD | 经典协议，广泛兼容 |
| VMess | TCP/WS/gRPC/H2 | 内置加密 | V2Ray 原生协议 |
| VLESS | TCP/WS/gRPC/H2 | TLS/Reality | 轻量级，无内置加密 |
| Trojan | TCP | TLS | TLS 伪装 |
| Trojan-Go | TCP | TLS | Trojan 增强版 |
| Snell | TCP | 内置加密 | Shadowsocks 增强 |
| TUIC | QUIC | QUIC 内置 | 基于 QUIC |
| Hysteria | QUIC | QUIC 内置 | 高速 QUIC 协议 |
| Hysteria2 | QUIC | QUIC 内置 | Hysteria 改进版 |
| WireGuard | UDP | WireGuard | VPN 协议 |
| SSH | TCP | SSH | SSH 隧道 |
| ShadowsocksR | TCP/UDP | AEAD | SS 增强版 |

## Web 面板（metacubexd）

metacubexd 是 mihomo 的官方 Web 面板，提供图形化的管理和监控界面：

- **节点管理** — 查看所有代理节点，切换代理组选中的节点
- **规则查看** — 查看当前生效的规则列表
- **连接监控** — 实时查看活跃连接，支持关闭单个连接
- **流量统计** — 实时上行/下行流量图表
- **日志查看** — 实时日志流
- **配置编辑** — 在线修改部分配置

部署方式：直接访问 mihomo 的 API 端口，或独立部署静态文件。

## 配置详解

mihomo 的 YAML 配置文件完整解析见 [[Mihomo-Clash-Config]]。

## 相关页面

- [[agent]] — 高性能协程代理引擎（服务端）
- [[docs/protocol/proxy-protocols]] — 代理协议
- [[docs/protocol/trojan-gfw]] — Trojan 协议实现
- [[Mihomo-Clash-Config]] — 配置文件详解
- [[client/mihomo-dns]] — mihomo DNS 配置
- [[client/tun]] — mihomo TUN 模式
