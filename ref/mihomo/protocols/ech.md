---
title: ECH
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [mihomo, protocol, ech]
---
# ECH（Encrypted Client Hello）

ECH（Encrypted Client Hello，加密客户端问候）是 TLS 1.3 的扩展，用于加密 TLS 握手初始阶段的 Client Hello 消息，防止域名（SNI）和其他握手参数在明文传输中泄露。它是 ESNI（Encrypted SNI）的继任者，提供了更全面的握手隐私保护。

## 技术概述

ECH 的核心目标是解决 TLS 握手中长期存在的隐私泄露问题。在 TLS 1.3 之前，即使应用数据被加密，Client Hello 消息仍然是明文传输的，其中包含：

- **SNI（Server Name Indication）**：客户端要访问的域名
- **支持的密码套件列表**：可用于指纹识别
- **ALPN 协议列表**：暴露客户端支持的应用层协议
- **其他扩展信息**：如 supported_groups、key_shares 等

ECH 特性：
- **加密 TLS Client Hello**：使用 HPKE（Hybrid Public Key Encryption）加密握手参数
- **防止 SNI 泄露**：中间人无法通过 SNI 得知目标域名
- **DNS HTTPS 记录查询**：通过 DNS HTTPS 记录（RFC 9460）动态获取 ECH 配置
- **Base64 静态配置**：也支持手动配置 ECH 公钥
- **向后兼容**：在不支持 ECH 的服务器上优雅降级

## ECH 原理

### 传统 TLS 握手的隐私问题

在标准 TLS 握手中，Client Hello 消息以明文形式传输：

```
Client                          Server
  |  Client Hello (明文)          |
  |  - SNI: example.com          |
  |  - Cipher Suites             |  ← GFW/ISP 可以看到 SNI
  |  - Key Share                 |
  |  - ALPN: h2, http/1.1        |
  |  - 其他扩展                   |
  |---------------------------->|
  |                              |
  |  Server Hello (加密开始)      |
  |<----------------------------|
  |  加密应用数据                 |
```

任何中间人（ISP、防火墙、WiFi 路由器）都可以通过检查 SNI 字段得知客户端要访问的域名。这是 TLS 协议长期以来的隐私漏洞。

### ECH 加密流程

ECH 通过以下方式解决该问题：

```
步骤一：获取 ECH 配置
Client                          DNS Server
  |  HTTPS 记录查询               |
  |  (_0rtt.example.com)         |
  |---------------------------->|
  |  返回 ECHConfigList          |
  |<----------------------------|

步骤二：加密 Client Hello
Client                          Server (外层)         Server (内层)
  |  Client Hello (外层/明文)     |                    |
  |  - SNI: 外层域名 (公共 SNI)   |                    |
  |  - ECH 扩展 (加密的内层)      |                    |
  |---------------------------->|                    |
  |                              | 解密 ECH 扩展       |
  |                              |-------------------->|
  |                              |                    | 获取真实 SNI
  |                              |                    | 处理内层 Client Hello
  |  Server Hello (加密)         |                    |
  |<----------------------------|                    |
  |  加密应用数据                 |                    |
```

### 外层 vs 内层

ECH 将 Client Hello 分为两个层次：

**外层（Outer Client Hello）**：
- 对网络可见，以明文传输
- 包含一个"公共 SNI"（通常是 CDN 域名或泛域名）
- 包含 ECH 扩展，该扩展包含加密的内层数据

**内层（Inner Client Hello）**：
- 使用 HPKE 加密
- 包含真实的 SNI（目标域名）
- 包含真实的密码套件、ALPN 等参数
- 只有持有对应私钥的服务器才能解密

## HPKE 加密流程

ECH 使用 HPKE（Hybrid Public Key Encryption，RFC 9180）来加密内层 Client Hello。HPKE 是一种混合公钥加密方案，结合了非对称加密的便利性和对称加密的高效性。

### HPKE 工作流程

```
1. 客户端从 ECHConfig 中获取服务器公钥 (pkS)

2. 客户端生成临时密钥对 (pkE, skE)

3. 使用 HPKE.SetupBaseS(pkS) 计算共享密钥:
   shared_secret = DH(skE, pkS)

4. 使用 HPKE.Seal 加密内层 Client Hello:
   encrypted_inner = HPKE.Seal(
       shared_secret,
       info = ECHConfig,
       aad = 外层 Client Hello,
       plaintext = 内层 Client Hello
   )

5. 将 pkE 和 encrypted_inner 编码为 ECH 扩展

6. 服务器使用私钥解密:
   shared_secret = DH(skS, pkE)
   inner_hello = HPKE.Open(shared_secret, ...)
```

### HPKE 参数

| 参数 | 值 | 说明 |
|------|-----|------|
| KEM（密钥封装机制） | DHKEM(X25519, HKDF-SHA256) | 使用 X25519 曲线 |
| KDF（密钥派生函数） | HKDF-SHA256 | 从共享密钥派生加密密钥 |
| AEAD（认证加密） | AES-128-GCM | 加密内层数据 |

HPKE 的安全优势：
- **前向安全**：每次握手使用不同的临时密钥对 (pkE, skE)
- **认证加密**：AEAD 确保加密数据的完整性和真实性
- **绑定外层**：AAD（Associated Authenticated Data）包含外层 Client Hello，防止外层被篡改

## 防止 SNI 泄露

### ECH 的防护层级

| 层级 | 传统 TLS | ECH |
|------|----------|-----|
| SNI | 明文可见 | 加密（内层） |
| ALPN | 明文可见 | 加密（内层） |
| Cipher Suites | 明文可见 | 外层显示公共配置 |
| Key Share | 明文可见 | 外层显示公共配置 |
| 扩展列表 | 明文可见 | 仅显示 ECH 扩展 |

### 外层 SNI 的作用

外层 SNI 是一个"公共"域名，通常是：
- CDN 服务的域名
- 服务器的默认域名
- 一个通用的泛域名

中间人只能看到外层 SNI，无法区分访问的是哪个具体网站（如果多个网站共享同一个 CDN 和 ECH 配置）。

### 与 ESNI 的区别

| 特性 | ESNI (旧) | ECH (新) |
|------|-----------|----------|
| 加密范围 | 仅 SNI 扩展 | 整个内层 Client Hello |
| 标准状态 | 已废弃 | IETF 标准 (RFC 9460, draft-ietf-tls-esni) |
| DNS 记录 | HTTPS RR 类型 | HTTPS RR 类型 (ech 字段) |
| 外层 SNI | 无 | 有（公共 SNI） |
| 兼容性 | 较差 | 更好（优雅降级） |

## mihomo 实现

### ECH 配置获取方式

mihomo 支持两种 ECH 配置获取方式：

**1. 静态配置（Base64）：**

```yaml
proxies:
  - name: "proxy-ech"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    ech-opts:
      enable: true
      config: base64-encoded-ech-config-list
```

**2. 动态 DNS 查询：**

```yaml
proxies:
  - name: "proxy-ech-dns"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    ech-opts:
      enable: true
      query-server-name: ech.example.com
```

动态查询时，mihomo 会向 DNS 服务器发送 HTTPS 类型记录查询：

```
查询: _0rtt.ech.example.com. IN HTTPS
响应: ech.example.com. IN HTTPS 1 . ech="base64-ech-config"
```

### ECH 优雅降级

当服务器不支持 ECH 时，mihomo 会自动降级到普通 TLS 握手：

```
ECH 连接尝试:
  |  Client Hello (外层 + ECH 扩展) |
  |------------------------------->|
  |                                 | 不支持 ECH
  |  HelloRetryRequest 或            |
  |  直接返回 Server Hello           |
  |<-------------------------------|
  |                                 |
  |  检测到不支持 ECH                |
  |  重连: 普通 TLS Client Hello    |
  |------------------------------->|
  |  正常 TLS 握手继续               |
```

### 支持的协议

ECH 可用于以下 mihomo 代理协议：

- [[vless]] - VLESS 代理
- [[vmess]] - VMess 代理
- [[trojan]] - Trojan 代理
- [[anytls]] - AnyTLS 代理
- [[hysteria]] - Hysteria 代理
- [[hysteria2]] - Hysteria2 代理
- [[tuic]] - TUIC 代理
- [[trusttunnel]] - TrustTunnel 代理

### ECH 配置字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `enable` | bool | 是 | 启用 ECH |
| `config` | string | 否 | Base64 编码的 ECHConfigList |
| `query-server-name` | string | 否 | DNS HTTPS 查询域名 |

## ECH 与 Reality 的结合

ECH 和 Reality 可以组合使用，提供更强的隐蔽性：

```yaml
proxies:
  - name: "vless-reality-ech"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    flow: xtls-rprx-vision
    reality-opts:
      public-key: public-key-base64url
      short-id: short-id-hex
    ech-opts:
      enable: true
      config: base64-ech-config
    client-fingerprint: chrome
```

组合效果：
1. **Reality** 处理外层 TLS 伪装（借用真实网站的 TLS 握手）
2. **ECH** 加密内层 Client Hello（隐藏真实 SNI）
3. 两层防护使流量指纹分析几乎不可能

## mihomo 实现位置

| 文件 | 描述 |
|------|------|
| `adapter/outbound/ech.go` | ECH 配置解析和集成 |
| `component/ech/ech.go` | ECH 核心实现，HPKE 加密/解密 |
| `dns/` | DNS HTTPS 记录查询支持 |

## 与 Prism 的兼容性

| Prism 模块 | 兼容性 | 说明 |
|------------|--------|------|
| Agent Worker | 完全兼容 | ECH 实现 |
| Pipeline | 完全兼容 | TLS 扩展 |
| Stealth | 完全兼容 | 防止 SNI 泄露 |
| Crypto | 完全兼容 | HPKE / X25519 集成 |

## YAML 配置示例

### 静态 ECH 配置

```yaml
proxies:
  - name: "proxy-ech"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    ech-opts:
      enable: true
      config: base64-encoded-ech-config-list
```

### 动态 DNS 查询

```yaml
proxies:
  - name: "proxy-ech-dns"
    type: vless
    server: server.example.com
    port: 443
    uuid: your-uuid
    tls: true
    ech-opts:
      enable: true
      query-server-name: ech.example.com
```

## 相关文档

- [[reality]] - Reality TLS 隐蔽
- [[core/stealth/ech|ech]] - ECH 技术详解
- [[core/crypto/x25519|x25519]] - X25519 密钥交换
- [[core/crypto/hkdf|hkdf]] - HKDF 密钥派生
