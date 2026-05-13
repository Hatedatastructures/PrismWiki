---
title: "代理协议总览"
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [protocol, proxy, overview, socks5, http, trojan, vless, shadowsocks]
related: ["[[http]]", "[[socks5]]", "[[trojan]]", "[[vless]]", "[[shadowsocks]]", "[[trojan-gfw]]", "[[pipeline]]", "[[protocol]]", "[[stealth]]", "[[agent]]"]
---

> 相关：[[http]] | [[socks5]] | [[trojan]] | [[vless]] | [[shadowsocks]] | [[trojan-gfw]] | [[pipeline]] | [[protocol]] | [[stealth]] | [[agent]]

# 代理协议总览

> Prism 支持的代理协议全景图，涵盖协议原理、实现差异与选型建议。

---

## 1. 什么是代理协议

代理协议定义了客户端与代理服务器之间交换连接请求和转发数据的规则。在对抗网络审查的场景中，代理协议不仅承担"告诉服务器去哪里"的功能，还需要在以下维度做出权衡：

| 维度 | 含义 |
|------|------|
| **加密** | 载荷是否对中间人不可读 |
| **认证** | 如何防止未授权用户访问代理 |
| **伪装** | 流量是否能冒充合法协议（如 HTTPS） |
| **性能** | 协议开销、是否支持多路复用、延迟 |
| **可探测性** | DPI 或主动探测能否识别出代理流量 |

---

## 2. Prism 支持的协议一览

### 2.1 HTTP 代理

最传统的代理形式。客户端发送 `CONNECT host:port HTTP/1.1` 请求，代理建立 TCP 隧道后中继数据。

- **优点**：兼容性极好，几乎所有客户端和浏览器原生支持
- **缺点**：明文传输（除非外套 TLS），易被 DPI 识别
- **Prism 实现**：[[http]] — 支持 CONNECT 和普通 HTTP 代理两种模式

### 2.2 SOCKS5

通用的 SOCKS 协议第五版（RFC 1928），支持 TCP 和 UDP 代理，以及用户名/密码认证。

- **优点**：标准化程度高，支持 UDP Associate
- **缺点**：协议握手特征明显（首字节 `0x05`），外套 TLS 后仍有指纹
- **Prism 实现**：[[socks5]] — 完整支持 CONNECT、UDP Associate 和 AUTH 子协商

### 2.3 Trojan

以 TLS 作为传输层的轻量代理协议，核心思想是"伪装成 HTTPS"。使用 SHA224 哈希密码认证，协议头部格式模仿 HTTP POST 请求体。

- **优点**：对 DPI 来说与正常 HTTPS 流量不可区分；CRLF 分隔的请求头简单可靠
- **缺点**：依赖有效 TLS 证书（或 Reality 等伪装方案）
- **Prism 实现**：[[trojan]] — C++23 协程实现，支持 CONNECT / UDP / MUX 三种命令
- **原始实现参考**：[[trojan-gfw]] — trojan-gfw/trojan 的协议规范与 Prism 实现的差异

### 2.4 VLESS

Xray 项目引入的无状态轻量级协议。与 Trojan 类似依赖外层 TLS，但使用 UUID 认证和纯二进制请求头（无 CRLF 文本分隔符）。

- **优点**：请求头更紧凑（最小 26 字节 vs Trojan 的 68 字节）；UUID 认证无哈希碰撞风险
- **缺点**：协议指纹比 Trojan 更难伪装（纯二进制）；不支持内置加密
- **Prism 实现**：[[vless]] — 支持 TCP / UDP / MUX 三种命令，AddnlInfoLen 必须为 0

### 2.5 Shadowsocks

基于 AEAD 加密的代理协议，最早由 clowwindy 于 2012 年发布。不依赖 TLS，自身提供加密和完整性保护。

- **优点**：无需证书管理；加密开销低（AES-256-GCM / ChaCha20-Poly1305）
- **缺点**：加密流量在统计层面有可识别特征；不支持 TLS 伪装
- **Prism 实现**：[[shadowsocks]] — 作为 fallback 协议，当 TLS 内层数据不匹配 Trojan/VLESS 时回退到此

---

## 3. 协议特性对比表

| 特性 | HTTP | SOCKS5 | Trojan | VLESS | Shadowsocks |
|------|------|--------|--------|-------|-------------|
| **内置加密** | 否 | 否 | 否（依赖 TLS） | 否（依赖 TLS） | 是（AEAD） |
| **认证方式** | 无/Basic | 无/USER-PASS | SHA224 密码哈希 | UUID (16B) | 密码派生密钥 |
| **TLS 依赖** | 可选 | 可选 | 必须 | 必须 | 否 |
| **伪装能力** | 差 | 差 | 强（冒充 HTTPS） | 中（二进制头） | 无 |
| **请求头最小字节** | ~30 | ~10 | 68 | 26 | 16+salt |
| **CRLF 文本分隔** | 是 | 否 | 是 | 否 | 否 |
| **UDP 支持** | 否 | 是 | 是（帧封装） | 是 | 是 |
| **MUX 支持** | 否 | 否 | 是（cmd=0x7F） | 是（cmd=0x7F） | 否 |
| **主动探测抗性** | 低 | 低 | 高 | 中 | 中 |
| **GFW 规避能力** | 低 | 低 | 高 | 高 | 中 |

---

## 4. Prism 中的协议处理流水线

所有代理协议在 Prism 内部遵循统一的处理链路，区别仅在检测阶段和协议解析层。

### 4.1 检测与分派

```
客户端连接
  |
  v
[session::diversion()] — 预读 24 字节
  |
  +-- 首字节 0x05          --> protocol_type::socks5
  +-- 首字节 0x16 + 0x03   --> protocol_type::tls
  +-- HTTP 方法 (GET/POST)  --> protocol_type::http
  +-- 其他                   --> protocol_type::shadowsocks (fallback)
  |
  v (如果是 TLS)
[伪装方案管道]
  Reality --> ShadowTLS --> 原生 TLS
  |
  v (TLS 解密后)
[detect_tls()] — 二次探测内层协议
  +-- HTTP 检查             --> http
  +-- VLESS 检查 (VER=0x00) --> vless
  +-- Trojan 检查 (hex+CRLF) --> trojan
  +-- 其他                   --> shadowsocks
```

详细检测逻辑参见 [[protocol]]。

### 4.2 Pipeline 层

检测完成后，`dispatch::handler_table` 将协议类型映射到对应的 pipeline 处理器：

```
protocol_type::socks5      --> pipeline::socks5()
protocol_type::http        --> pipeline::http()
protocol_type::trojan      --> pipeline::trojan()
protocol_type::vless       --> pipeline::vless()
protocol_type::shadowsocks --> pipeline::shadowsocks()
```

每个 pipeline 处理器执行：`wrap_with_preview` → `make_relay` → `handshake` → `forward/associate/bootstrap`。详见 [[pipeline]]。

### 4.3 统一的传输抽象

所有协议共享同一套 `transmission` 接口（`async_read_some` / `async_write_some`），使得协议层无需关心底层是原生 TCP、TLS 流还是 AEAD 加密流。

---

## 5. 协议选型建议

| 场景 | 推荐协议 | 原因 |
|------|----------|------|
| 高审查环境（GFW 深度检测） | Trojan + Reality | TLS 伪装 + 伪造证书，主动探测抗性最强 |
| 一般审查环境 | VLESS + TLS | 请求头紧凑，UUID 认证简单 |
| 无需 TLS（自建加密通道） | Shadowsocks | 无需证书，部署简单 |
| 兼容老旧客户端 | SOCKS5 / HTTP | 原生支持，无需特殊客户端 |
| 高并发低延迟 | Trojan / VLESS + MUX | 支持 smux/yamux 多路复用，减少 TLS 握手开销 |

---

## 6. 原始协议实现参考

Prism 中的协议实现均参考了各协议的原始开源项目：

| 协议 | 原始项目 | Prism 实现差异 |
|------|----------|----------------|
| Trojan | [trojan-gfw/trojan](https://github.com/trojan-gfw/trojan) | C++23 协程、PMR 内存、MUX 扩展。详见 [[trojan-gfw]] |
| VLESS | [xtls/xray-core](https://github.com/XTLS/Xray-core) | 不支持 Flow/XTLS，仅 plain VLESS |
| Shadowsocks | [shadowsocks/shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev) | 作为 fallback 协议，非独立使用 |
| SOCKS5 | RFC 1928 | 完整实现，含 UDP Associate |
| HTTP CONNECT | RFC 7231 | 支持 CONNECT 隧道和普通代理 |

---

## 7. 相关页面

- [[http]] — HTTP 代理协议详解
- [[socks5]] — SOCKS5 协议详解
- [[trojan]] — Prism 中的 Trojan 协议实现
- [[trojan-gfw]] — Trojan-GFW 原始协议规范
- [[vless]] — VLESS 协议规范
- [[shadowsocks]] — Shadowsocks 协议规范
- [[pipeline]] — 协议流水线架构
- [[protocol]] — 协议检测与解析实现细节
- [[stealth]] — 伪装方案总览（Reality / ShadowTLS / Restls）
- [[agent]] — Agent 模块概览
