---
title: Prism 知识库索引
layer: root
---

# Prism 知识库

> Prism 高性能代理引擎的项目文档 — 四层分离架构：core（详细）、dev（开发）、docs（用户）、ref（参考）
> 最后更新：2026-05-17

---

## 知识库架构

采用四层分离架构，避免知识重复：

| 层级 | 目录 | 详细程度 | 职责 |
|------|------|----------|------|
| 模块实现层 | [[core/overview|core]] | 详细 | 函数实现、调用链、状态变化 |
| 开发排障层 | [[dev/overview|dev]] | 详细 | 编码规范、排障方法、Bug记录 |
| 使用指南层 | [[docs/overview|docs]] | 简单 | 用户文档、部署、配置 |
| 参考知识层 | [[ref/overview|ref]] | 中等 | 协议规范、加密原理、mihomo资料 |

---

## Core 层 — 模块实现细节

> 详见 [[core/overview|Core 层总览]]

### 架构总览

- [[core/architecture|六阶段流水线架构]]
- [[core/startup|启动流程详解]]
- [[core/flow|协议处理流程详解]]
- [[core/infrastructure|基础设施总览]]

### 模块列表

| 模块 | 职责 | 源码位置 |
|------|------|----------|
| [[core/agent/overview|Agent]] | 前端监听、会话管理 | `include/prism/agent/` |
| [[core/channel/overview|Channel]] | 连接池、传输层 | `include/prism/channel/` |
| [[core/pipeline/overview|Pipeline]] | 协议处理器 | `include/prism/pipeline/` |
| [[core/protocol/overview|Protocol]] | 协议格式、常量 | `include/prism/protocol/` |
| [[core/stealth/overview|Stealth]] | TLS 伪装 | `include/prism/stealth/` |
| [[core/recognition/overview|Recognition]] | 协议识别 | `include/prism/recognition/` |
| [[core/resolve/overview|Resolve]] | DNS 解析 | `include/prism/resolve/` |
| [[core/multiplex/overview|Multiplex]] | 多路复用 | `include/prism/multiplex/` |
| [[core/crypto/overview|Crypto]] | 加密模块 | `include/prism/crypto/` |
| [[core/outbound/overview|Outbound]] | 出站代理 | `include/prism/outbound/` |

### 基础设施

| 模块 | 职责 |
|------|------|
| [[core/memory/overview|Memory]] | PMR 内存池 |
| [[core/fault/overview|Fault]] | 错误码 |
| [[core/exception/overview|Exception]] | 异常体系 |
| [[core/trace/overview|Trace]] | 日志系统 |
| [[core/transformer/overview|Transformer]] | JSON 序列化 |
| [[core/loader/overview|Loader]] | 配置加载 |

---

## Dev 层 — 开发规范与排障

> 详见 [[dev/overview|Dev 层总览]]

### 编码规范

- [[dev/coding/naming|命名规范]]
- [[dev/coding/coroutine|协程约定]]
- [[dev/coding/pmr|PMR 使用规范]]
- [[dev/coding/doxygen|注释规范]]
- [[dev/coding/lifecycle|生命周期安全]]
- [[dev/coding/error|错误处理策略]]

### 测试体系

- [[dev/testing/framework|测试框架]]
- [[dev/testing/commands|测试命令]]
- [[dev/testing/benchmark|基准测试]]
- [[dev/testing/stress|压力测试]]

### 排障方法

- [[dev/debugging/connection|连接问题排查]]
- [[dev/debugging/protocol|协议问题排查]]
- [[dev/debugging/memory|内存问题排查]]
- [[dev/debugging/performance|性能问题排查]]
- [[dev/debugging/tls|TLS 问题排查]]
- [[dev/debugging/log-analysis|日志分析方法]]

#### 深层故障分析

- [[dev/debugging/deep-dive/protocol-boundaries|协议实现边界]] — SS2022/Trojan/VLESS/SOCKS5 深层边界
- [[dev/debugging/deep-dive/reality-handshake|Reality 握手故障]] — 12 步认证 + 5 阶段故障
- [[dev/debugging/deep-dive/shadowtls-limitations|ShadowTLS 限制]] — 竞态、HMAC 退化
- [[dev/debugging/deep-dive/stealth-limitations|伪装方案限制]] — rewind、空壳方案
- [[dev/debugging/deep-dive/recognition-weaknesses|识别弱点]] — 误判、弱特征
- [[dev/debugging/deep-dive/system-risks|系统级风险]] — 资源耗尽、无优雅关闭
- [[dev/debugging/deep-dive/multiplex-boundaries|多路复用边界]] — pending、资源上限
- [[dev/debugging/log-reference|日志字典]] — 完整日志消息与错误码索引

### 构建

- [[dev/building/cmake|CMake 结构]]
- [[dev/building/commands|构建命令]]
- [[dev/building/dependencies|依赖管理]]

### 扩展开发

- [[dev/extending/protocol|新协议集成]]
- [[dev/extending/stealth|新伪装方案]]

### Bug 记录

- [[dev/bugs/template|Bug 报告模板]]

### 开发路线图

- [[dev/roadmap|开发路线图]]

---

## Docs 层 — 使用指南

> 详见 [[docs/overview|Docs 层总览]]

### 快速开始

- [[docs/getting-started|快速开始]]
- [[docs/deployment|部署指南]]
- [[docs/configuration|配置说明]]
- [[docs/client-setup|客户端配置]]

### 故障排查

- [[docs/troubleshooting|故障排查]]
- [[docs/faq|常见问题]]
- [[docs/known-issues|已知问题清单]]

### 其他

- [[docs/upgrade|升级指南]]
- [[docs/security|安全注意事项]]
- [[docs/performance-tips|性能建议]]

---

## Ref 层 — 参考知识

> 详见 [[ref/overview|Ref 层总览]]

### Mihomo 完整参考

> 详见 [[ref/mihomo/index|Mihomo 总索引]]

#### 出站协议（22个）

- [[ref/mihomo/protocols/socks5|SOCKS5]]
- [[ref/mihomo/protocols/socks4|SOCKS4]]
- [[ref/mihomo/protocols/http|HTTP Proxy]]
- [[ref/mihomo/protocols/trojan|Trojan]]
- [[ref/mihomo/protocols/vless|VLESS]]
- [[ref/mihomo/protocols/vmess|VMess]]
- [[ref/mihomo/protocols/shadowsocks|Shadowsocks]]
- [[ref/mihomo/protocols/shadowsocksr|ShadowsocksR]]
- [[ref/mihomo/protocols/snell|Snell]]
- [[ref/mihomo/protocols/ssh|SSH Tunnel]]
- [[ref/mihomo/protocols/hysteria|Hysteria]]
- [[ref/mihomo/protocols/hysteria2|Hysteria2]]
- [[ref/mihomo/protocols/tuic|TUIC]]
- [[ref/mihomo/protocols/wireguard|WireGuard]]
- [[ref/mihomo/protocols/anytls|AnyTLS]]
- [[ref/mihomo/protocols/reality|Reality]]
- [[ref/mihomo/protocols/ech|ECH]]
- [[ref/mihomo/protocols/trusttunnel|TrustTunnel]]
- [[ref/mihomo/protocols/mieru|Mieru]]
- [[ref/mihomo/protocols/masque|MASQUE]]
- [[ref/mihomo/protocols/sudoku|Sudoku]]

#### 传输层插件

- [[ref/mihomo/transport/shadowtls|ShadowTLS]]
- [[ref/mihomo/transport/restls|Restls]]
- [[ref/mihomo/transport/v2ray-plugin|V2Ray Plugin]]
- [[ref/mihomo/transport/gun|gRPC (gun)]]

#### 配置系统

- [[ref/mihomo/proxy-groups/overview|代理组]]
- [[ref/mihomo/rules/overview|规则系统]]
- [[ref/mihomo/dns/overview|DNS 配置]]
- [[ref/mihomo/tun/overview|TUN 模式]]
- [[ref/mihomo/config/overview|配置模板]]

#### 兼容性

- [[ref/mihomo/compatibility/prism|Prism 兼容性对照]]

### 加密知识

- [[ref/crypto/aead-basics|AEAD 原理]]
- [[ref/crypto/key-exchange|密钥交换原理]]
- [[ref/crypto/hkdf-theory|HKDF 理论]]

### 协议知识

- [[ref/protocol/tls-handshake|TLS 握手流程]]
- [[ref/protocol/socks5-spec|SOCKS5 RFC]]
- [[ref/protocol/tcp-basics|TCP 基础]]
- [[ref/protocol/quic-basics|QUIC 基础]]

### 网络知识

- [[ref/network/happy-eyeballs|Happy Eyeballs]]
- [[ref/network/dns-resolution|DNS 解析原理]]
- [[ref/network/gfw|GFW 原理]]

### 编程知识

- [[ref/programming/boost-asio|Boost.Asio 协程]]
- [[ref/programming/cpp23-coroutine|C++23 协程]]
- [[ref/programming/pmr-concepts|PMR 概念]]

### 术语表

- [[ref/glossary|术语表]]

---

## Skills

- [[skills/prism-wiki-sync/SKILL|wiki-sync skill]] — 知识库同步检查

---

## 规范

- [[SCHEMA|知识库规范]]