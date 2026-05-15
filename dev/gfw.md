---
title: "GFW 封锁原理"
created: 2026-05-13
updated: 2026-05-13
type: dev
tags: [gfw, censorship, detection, network]
related:
  - "[[stealth]]"
  - "[[stealth/reality]]"
  - "[[docs/stealth/proxy-detection]]"
  - "[[docs/protocol/proxy-protocols]]"
  - "[[docs/protocol/trojan-gfw]]"
  - "[[dev/tls]]"
  - "[[agent]]"
---

# GFW 封锁原理

Great Firewall（GFW）是中国的互联网审查系统，通过多种技术手段检测和封锁翻墙流量。理解 GFW 的工作原理是设计抗审查代理协议的基础。

## 检测方法概览

GFW 使用多种互补的检测技术：

| 方法 | 层级 | 特点 |
|------|------|------|
| IP 封锁 | 网络层 | 简单粗暴，易绕过 |
| DNS 污染 | 应用层 | 针对域名解析 |
| TCP Reset | 传输层 | 主动中断连接 |
| 深度包检测 | 应用层 | 分析流量内容 |
| TLS 指纹 | 应用层 | 识别客户端特征 |
| 主动探测 | 应用层 | 模拟客户端探测服务器 |

## IP 封锁

最简单的封锁方式，将已知代理服务器 IP 加入黑名单。

### 实现方式

- 路由黑洞：BGP 层面丢弃目标 IP 的路由
- 防火墙丢弃：在网络层直接丢弃数据包
- DNS 返回错误 IP（与 DNS 污染配合）

### 特征

- 对该 IP 的所有连接都会失败
- 从中国境内 ping 该 IP 无响应
- 从境外可以正常访问

### 绕过

- 使用大量 IP 轮换（Cloudflare 等 CDN 的 IP 池）
- 使用域名前置（Domain Fronting）
- 使用 CDN 作为中转

## DNS 污染

GFW 在 DNS 层面劫持特定域名的解析。

### 工作原理

```
Client                     GFW                     DNS Server
  |                         |                         |
  |--- DNS Query ---------->|                         |
  |    (google.com)         |--- DNS Query ---------->|
  |                         |<-- 正确结果 -------------|
  |<-- 伪造结果 ------------|  (被丢弃)
  |    (错误 IP)            |
```

GFW 检测到 DNS 查询包含敏感域名后，立即返回伪造的 DNS 响应，比真实响应更快到达客户端。

### 特征

- 仅影响特定域名
- 使用加密 DNS（DoH/DoT）可绕过
- 返回的 IP 通常是固定的错误地址

### 绕过

- 使用 DoH/DoT（DNS over HTTPS/TLS）
- 使用境外 DNS 服务器（但 GFW 可能封锁 DNS 服务器 IP）
- 在代理客户端中硬编码 IP 或使用预置的 IP 列表

## TCP Reset（RST 注入）

GFW 主动发送伪造的 TCP RST 包中断连接。

### 工作原理

1. GFW 检测到连接中包含敏感内容
2. 向连接双方发送伪造的 RST 包
3. 双方收到 RST 后断开连接

### 触发条件

- HTTP 请求中包含敏感关键词
- SNI 中包含敏感域名（TLS 连接）
- 连接模式匹配已知代理协议特征

### 绕过

- 使用 TLS 加密（阻止内容检测）
- 忽略 RST 包（需要内核层面修改）
- 使用无状态协议（如 QUIC，不受 TCP RST 影响）

## 深度包检测（DPI）

GFW 最强大的检测手段，分析数据包内容识别代理流量。

### 检测维度

1. **协议特征**：识别协议握手模式（如 SS 的加密握手、VMess 的认证头）
2. **流量统计**：分析包大小分布、时间间隔、突发模式
3. **行为分析**：连接模式、持续时间、数据量

### 协议特征检测

早期代理协议有明显的协议特征：

- **Shadowsocks**：加密握手模式可被机器学习识别
- **VMess**：认证头的熵值分布异常
- **OpenVPN**：TLS 握手中的特定扩展和证书特征

### 流量统计检测

即使加密，流量的元数据仍然暴露信息：

- 包大小分布：视频流 vs 网页浏览 vs 代理隧道有不同模式
- 时间间隔：交互式应用 vs 批量传输的模式差异
- 突发行为：TLS 握手后的数据传输模式

详见 [[docs/stealth/proxy-detection]]。

## TLS 指纹检测

GFW 分析 TLS ClientHello 的指纹特征（见 [[dev/tls]]）。

### JA3/JA4 指纹

- 浏览器的 TLS 指纹是已知的（Chrome、Firefox、Safari 各有固定指纹）
- 代理工具的 TLS 库（如 Go 的 crypto/tls）产生不同的指纹
- 如果 TLS 指纹与 SNI 声称的网站不匹配，可能被标记

### JARM 指纹

服务端指纹检测：

- 发送特制的 ClientHello 探测包
- 分析服务端 TLS 响应的差异
- 生成服务器指纹，与已知代理服务器匹配

### 绕过

- 使用浏览器内核进行 TLS（如 uTLS 模拟浏览器指纹）
- Reality 技术：让 TLS 握手看起来与真实网站一致（见 [[stealth/reality]]）
- 使用与主流网站相同的 TLS 配置

## 主动探测

GFW 会主动连接可疑的服务器，验证其是否为代理。

### 探测方式

1. **连接探测**：发起 TCP 连接，观察响应
2. **协议探测**：发送特定协议数据，观察响应
3. **TLS 探测**：完成 TLS 握手，检查证书内容

### 探测识别

- 如果服务器像正常网站一样响应 → 不是代理
- 如果服务器返回代理协议特有响应 → 确认为代理
- 如果服务器拒绝连接 → 可疑但不确定

### 绕过

- **Reality**：探测连接返回真实网站内容（见 [[stealth/reality]]）
- **ShadowTLS**：类似原理，将探测流量转发给真实 TLS 服务器
- **回落机制**：非代理流量回落到正常网站

## 代理协议的演进

GFW 的检测能力推动了代理协议的不断演进：

```
SS (2012) → SSR (2015) → VMess (2015) → Trojan (2019) → VLESS (2020) → Reality (2022)
```

### 各阶段的对抗关系

| 协议 | GFW 检测方式 | 绕过手段 |
|------|-------------|----------|
| SS | 流量统计 + 机器学习 | 协议混淆 |
| SSR | 协议特征 | 仍可被深度分析 |
| VMess | 认证头特征 | 动态端口 |
| Trojan | 伪装 HTTPS | 需要真实域名/证书 |
| VLESS | 轻量但依赖 TLS | 配合 Reality |
| Reality | 主动探测 + 指纹 | 返回真实网站内容 |

详见 [[docs/protocol/trojan-gfw]] 中 Trojan 协议对抗 GFW 的详细分析。

## Prism 的应对策略

Prism 在设计时充分考虑了 GFW 的检测能力：

1. **TLS 层**：使用 BoringSSL 精细控制 ClientHello，降低 TLS 指纹特征
2. **Reality 支持**：原生支持 Reality TLS，无需域名和证书
3. **流量伪装**：模拟正常 HTTPS 流量的统计特征
4. **模块化设计**：stealth 模块独立于协议层，可灵活更新对抗策略

## 参考资源

- [GFW 技术分析报告](https://gfw.report/)
- [TLS 指纹检测研究](https://tlsfingerprint.io/)
- [OONI 网络审查观测](https://ooni.org/)

## 相关页面

- [[stealth]] — 隐蔽技术概览
- [[stealth/reality]] — Reality TLS 技术
- [[docs/stealth/proxy-detection]] — 代理检测技术详解
- [[docs/protocol/proxy-protocols]] — 代理协议概览
- [[docs/protocol/trojan-gfw]] — Trojan 协议与 GFW 对抗
- [[dev/tls]] — TLS 协议基础
- [[agent]] — Prism Agent 概览
