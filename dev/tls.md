---
title: "TLS 协议基础"
created: 2026-05-13
updated: 2026-05-13
type: dev
tags: [tls, encryption, security, certificate]
related:
  - "[[stealth/reality]]"
  - "[[docs/stealth/pki-certificates]]"
  - "[[crypto]]"
  - "[[dev/tcp]]"
  - "[[protocol/trojan]]"
  - "[[protocol/vless]]"
  - "[[agent]]"
---

# TLS 协议基础

TLS（Transport Layer Security）是互联网加密通信的基石。在代理领域，TLS 不仅保护数据传输，还是对抗审查的关键伪装手段。

## TLS 1.3 握手流程

TLS 1.3 将握手简化为 1-RTT（往返延迟）：

```
Client                                Server
  |                                     |
  |--- ClientHello -------------------->|  支持的密码套件、密钥共享
  |    (key_share, supported_versions)  |
  |                                     |
  |<-- ServerHello ---------------------|  选定密码套件、密钥共享
  |    {EncryptedExtensions}            |  加密的扩展信息
  |    {Certificate*}                   |  服务器证书
  |    {CertificateVerify*}             |  证书签名
  |    {Finished}                       |  握手完成验证
  |                                     |
  |--- {Finished} --------------------->|  客户端确认
  |                                     |
  |     应用数据加密传输                  |
```

关键改进（相比 TLS 1.2）：

- 握手从 2-RTT 减少到 1-RTT
- 支持 0-RTT 恢复（Early Data）
- 移除不安全的密码套件（RSA 密钥交换、CBC 模式等）
- 加密更多握手信息（证书等）

### 0-RTT 早期数据

TLS 1.3 允许在恢复连接时发送早期数据：

```
Client                                Server
  |--- ClientHello + EarlyData -------->|  0-RTT
  |<-- ServerHello + Finished ----------|
  |--- Finished + 应用数据 ------------>|  1-RTT
```

风险：早期数据可能被重放。代理协议中需谨慎使用。

## 证书链与 PKI

TLS 依赖公钥基础设施（PKI）验证服务器身份：

```
Root CA
  └── Intermediate CA
        └── Server Certificate (*.example.com)
```

### 验证过程

1. 客户端收到服务器证书链
2. 从根 CA 开始逐级验证签名
3. 检查证书域名是否匹配
4. 检查证书是否过期
5. 检查证书是否被吊销（CRL/OCSP）

详见 [[docs/stealth/pki-certificates]]。

## TLS 指纹

### JA3 指纹

JA3 从 ClientHello 中提取参数生成指纹：

```
JA3 = TLSVersion,Ciphers,Extensions,EllipticCurves,EllipticCurvePointFormats
```

对提取的字段用逗号连接后取 MD5 哈希。不同浏览器/库的 JA3 指纹不同，可用于识别客户端类型。

### JA4 指纹

JA4 是 JA3 的改进版本，提供更精细的分类：

```
JA4 = TLSVersion_ALPN_CipherCount_ExtensionsCount_SNI_SNIExt
```

### JARM 指纹

JARM 从服务端角度指纹化 TLS 服务器。通过发送 10 个特制的 ClientHello，分析服务端响应的差异，生成服务器指纹。

### 对代理的影响

GFW 使用 TLS 指纹识别代理服务器。如果代理服务器的 TLS 指纹与常见网站不同，可能被标记为可疑。

## BoringSSL

Prism 使用 BoringSSL（Google 维护的 OpenSSL 分支）作为 TLS 实现：

- 更活跃的安全维护
- 更小的代码体积
- 支持 TLS 1.3 的所有现代特性
- 可以精细控制 ClientHello 扩展，降低 TLS 指纹特征

BoringSSL 相比 OpenSSL 的优势：

| 特性 | BoringSSL | OpenSSL |
|------|-----------|---------|
| TLS 1.3 支持 | 原生 | 需要 1.1.1+ |
| 代码审计 | Google 持续审计 | 社区维护 |
| API 简洁性 | 更现代 | 历史包袱重 |
| 0-RTT 支持 | 完善 | 有限 |

## TLS 在代理协议中的应用

### Trojan 协议

[[protocol/trojan]] 的核心思想是"伪装成 HTTPS 流量"：

```
[Client] ---TLS (正常 HTTPS)--- [Trojan Server]
         ClientHello 中包含真实域名（SNI）
         TLS 内部传输代理数据
```

Trojan 服务器收到连接后：
1. 完成 TLS 握手（像正常 HTTPS 服务器一样）
2. 读取第一个包，判断是 Trojan 协议还是正常 HTTP
3. 如果是 Trojan，解析代理指令；否则转发为正常 HTTP

### VLESS over TLS

[[protocol/vless]] 通常与 TLS 配合使用：

```
[Client] ---TLS--- [VLESS Server]
         TLS 内部：VLESS Header + 代理数据
```

VLESS 不加密（依赖 TLS），因此更轻量。

## Reality TLS

Reality（见 [[stealth/reality]]）是革命性的 TLS 伪装技术：

### 标准 TLS 的问题

即使使用 TLS，握手阶段的 ServerHello 会暴露代理服务器的身份（证书内容与真实网站不同）。

### Reality 的解决方案

1. 客户端使用真实网站的证书信息构建 ClientHello
2. 服务端用自己的私钥完成握手，但不发送自签名证书
3. 如果被中间人检测（主动探测），回复真实网站的正常响应

关键技术：

- **x25519 密钥交换**：服务端和客户端使用预共享公钥完成密钥协商
- **Short ID**：区分合法连接和探测连接
- **伪装目标**：选择 TLS 指纹相似的真实网站

Reality 的优势：
- 服务端无需域名和证书
- 无法通过主动探测识别
- TLS 握手看起来与访问真实网站完全一致

## TLS 最佳实践

### 客户端配置

- 强制 TLS 1.3
- 使用强密码套件（AES-256-GCM, ChaCha20-Poly1305）
- 启用证书固定（Certificate Pinning）防止 MITM

### 服务端配置

- 使用有效的证书（Let's Encrypt 或 Reality）
- 启用 OCSP Stapling 加快握手
- 配置 HSTS 头部

### 代理场景

- 选择与主流网站相似的 TLS 配置，降低指纹特征
- 使用 Reality 或 ShadowTLS 进一步伪装
- 避免自签名证书（容易被识别）

## 相关页面

- [[stealth/reality]] — Reality TLS 伪装技术
- [[docs/stealth/pki-certificates]] — PKI 证书体系
- [[crypto]] — 密码学概览
- [[dev/tcp]] — TCP 协议基础
- [[protocol/trojan]] — Trojan 协议
- [[protocol/vless]] — VLESS 协议
- [[agent]] — Prism Agent 概览
