---
title: "TLS 1.3 协议"
category: "protocol"
type: ref
module: ref
source: "RFC 8446"
tags: [协议, tls, 握手, 加密, 安全, 传输层安全]
created: 2026-05-15
updated: 2026-05-17
related: [tls-clienthello, tls-serverhello, tls-extensions, hkdf, aes-gcm, x25519]
---

# TLS 1.3 协议

**类别**: 协议

## 概述

TLS 1.3（Transport Layer Security version 1.3）是传输层安全协议的最新版本，由 IETF 于 2018 年 8 月发布为 RFC 8446。TLS 1.3 相比 TLS 1.2 进行了重大改进，简化了握手流程，移除了不安全的密码套件和特性，强制实现了前向保密（Forward Secrecy），并显著提升了安全性和性能。

### 发展历程

TLS 协议的发展经历了多个版本：

- **SSL 1.0** (1995)：Netscape 开发，从未公开发布，存在严重安全漏洞
- **SSL 2.0** (1995)：公开发布，后被发现存在多处安全缺陷
- **SSL 3.0** (1996)：改进版本，但仍有设计缺陷（如 POODLE 攻击）
- **TLS 1.0** (1999, RFC 2246)：IETF 标准化，基于 SSL 3.0
- **TLS 1.1** (2006, RFC 4346)：添加对 CBC 攻击的防护
- **TLS 1.2** (2008, RFC 5246)：引入 SHA-256、AEAD 密码套件
- **TLS 1.3** (2018, RFC 8446)：重大安全改进和性能优化

### 核心设计目标

TLS 1.3 的设计遵循以下核心目标：

1. **安全性优先**：移除所有已知不安全的密码套件和特性
2. **隐私保护**：加密尽可能多的握手消息
3. **性能优化**：减少握手往返次数，支持 0-RTT 数据
4. **简化实现**：减少配置选项，降低实现复杂度
5. **前向保密**：强制使用临时密钥交换，确保会话密钥泄露不影响历史通信

### 与 TLS 1.2 的关键差异

TLS 1.3 相比 TLS 1.2 有以下关键变化：

| 特性 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| 握手往返 | 2-RTT | 1-RTT |
| 密钥交换 | RSA/DH/ECDH | 仅 (EC)DHE |
| 前向保密 | 可选 | 强制 |
| 密码套件数量 | 37+ | 5 |
| 握手消息加密 | 部分加密 | 全部加密（除 ClientHello/ServerHello） |
| 0-RTT 支持 | 不支持 | 支持 |
| 压缩 | 支持 | 移除 |
| 重协商 | 支持 | 移除 |
| 静态 RSA/DH | 支持 | 移除 |

## 协议详解

### 握手流程

#### 1-RTT 握手

TLS 1.3 最显著的变化是将握手从 2-RTT 减少到 1-RTT。客户端在第一条消息中就发送密钥交换材料，服务器在响应中完成密钥交换。

```
Client                                    Server
  |                                          |
  |---------- ClientHello ------------------>|
  |          (supported_versions: 0x0304)   |
  |          (supported_groups: x25519)      |
  |          (key_share: x25519公钥)         |
  |          (signature_algorithms)          |
  |          (cipher_suites)                 |
  |                                          |
  |<--------- ServerHello -------------------|
  |          (supported_version: 0x0304)    |
  |          (key_share: x25519公钥)         |
  |          (cipher_suite: TLS_AES_128...) |
  |                                          | ← 此时双方已计算出握手密钥
  |<--------- EncryptedExtensions -----------| ← 开始加密
  |          (server_name, alpn...)          |
  |                                          |
  |<--------- Certificate -------------------|
  |          (服务器证书链)                   |
  |                                          |
  |<--------- CertificateVerify -------------|
  |          (证书签名验证)                   |
  |                                          |
  |<--------- Finished ----------------------|
  |          (握手完成验证)                   |
  |                                          | ← 此时双方已计算出应用密钥
  |---------- Finished --------------------->|
  |          (握手完成验证)                   |
  |                                          |
  |<========= Application Data ==============>| ← 应用数据传输
```

#### 密钥派生阶段

TLS 1.3 使用 HKDF (HMAC-based Key Derivation Function) 进行密钥派生，分为三个阶段：

```
阶段 1: Early Secret（预共享密钥场景）
┌─────────────────────────────────────────────┐
│ Early Secret = HKDF-Extract(0, PSK)        │
│ 或                                          │
│ Early Secret = HKDF-Extract(0, 0^hash_len)│
└─────────────────────────────────────────────┘

阶段 2: Handshake Secret（握手密钥）
┌─────────────────────────────────────────────┐
│ Shared Secret = ECDH(ClientPrivate,        │
│                     ServerPublic)          │
│ Handshake Secret = HKDF-Extract(           │
│     Derive-Secret(Early Secret,            │
│                   "derived", ""),         │
│     Shared Secret)                         │
└─────────────────────────────────────────────┘

阶段 3: Master Secret（主密钥）
┌─────────────────────────────────────────────┐
│ Master Secret = HKDF-Extract(             │
│     Derive-Secret(Handshake Secret,       │
│                   "derived", ""),         │
│     0^hash_len)                            │
└─────────────────────────────────────────────┘
```

#### 0-RTT 握手（预共享密钥）

对于有预共享密钥（PSK）的场景，TLS 1.3 支持 0-RTT 握手，客户端可以在第一条消息中发送加密的应用数据：

```
Client                                    Server
  |                                          |
  |---------- ClientHello ------------------>|
  |          (pre_shared_key)               |
  |          (early_data)                   |
  |========== Early Data ===================>|
  |          (加密的应用数据)                 |
  |                                          |
  |<--------- ServerHello -------------------|
  |          (pre_shared_key)               |
  |                                          |
  |<--------- EncryptedExtensions -----------|
  |                                          |
  |<--------- Finished ----------------------|
  |                                          |
  |---------- Finished --------------------->|
  |                                          |
  |<========= Application Data ==============>|
```

### 密码套件

TLS 1.3 大幅简化了密码套件，仅支持 5 个 AEAD 密码套件：

| 密码套件 | 值 | 密钥交换 | 认证 | 加密 | 哈希 |
|----------|-----|----------|------|------|------|
| TLS_AES_128_GCM_SHA256 | 0x1301 | ECDHE/PSK | 签名/PSK | AES-128-GCM | SHA-256 |
| TLS_AES_256_GCM_SHA384 | 0x1302 | ECDHE/PSK | 签名/PSK | AES-256-GCM | SHA-384 |
| TLS_CHACHA20_POLY1305_SHA256 | 0x1303 | ECDHE/PSK | 签名/PSK | ChaCha20-Poly1305 | SHA-256 |
| TLS_AES_128_CCM_SHA256 | 0x1304 | ECDHE/PSK | 签名/PSK | AES-128-CCM | SHA-256 |
| TLS_AES_128_CCM_8_SHA256 | 0x1305 | ECDHE/PSK | 签名/PSK | AES-128-CCM-8 | SHA-256 |

**密码套件选择原则**：
- AES-GCM 适合有 AES-NI 指令集的 x86_64 CPU
- ChaCha20-Poly1305 适合 ARM 或无硬件加速的环境
- AES-CCM 和 AES-CCM-8 用于资源受限设备

### 密钥交换

TLS 1.3 支持以下密钥交换算法（通过 supported_groups 扩展协商）：

| 名称 | 值 | 类型 | 密钥大小 |
|------|-----|------|----------|
| secp256r1 (P-256) | 0x0017 | NIST 曲线 | 256-bit |
| secp384r1 (P-384) | 0x0018 | NIST 曲线 | 384-bit |
| secp521r1 (P-521) | 0x0019 | NIST 曲线 | 521-bit |
| x25519 | 0x001D | Curve25519 | 256-bit |
| x448 | 0x001E | Curve448 | 448-bit |
| ffdhe2048 | 0x0100 | 有限域 | 2048-bit |
| ffdhe3072 | 0x0101 | 有限域 | 3072-bit |
| ffdhe4096 | 0x0102 | 有限域 | 4096-bit |
| ffdhe6144 | 0x0103 | 有限域 | 6144-bit |
| ffdhe8192 | 0x0104 | 有限域 | 8192-bit |

**推荐使用**：x25519（性能与安全兼顾）或 x448（更高安全性）

### 记录层

TLS 1.3 记录层格式：

```
TLS Record:
┌────────────────────────────────────────────┐
│ ContentType: 1 byte                        │
│   - 0x14: ChangeCipherSpec (遗留)          │
│   - 0x15: Alert                            │
│   - 0x16: Handshake                        │
│   - 0x17: Application Data                 │
├────────────────────────────────────────────┤
│ LegacyRecordVersion: 2 bytes              │
│   - 通常为 0x0303 (TLS 1.2 兼容)           │
├────────────────────────────────────────────┤
│ Length: 2 bytes                            │
│   - 最大 2^14 + 256 字节                   │
├────────────────────────────────────────────┤
│ Fragment: variable                         │
│   - 加密后为 AEAD 密文                     │
└────────────────────────────────────────────┘

加密后的记录：
┌────────────────────────────────────────────┐
│ Ciphertext: Length - AEAD tag bytes        │
├────────────────────────────────────────────┤
│ AEAD Tag: 16 bytes (AES-GCM) 或           │
│           16 bytes (ChaCha20-Poly1305)    │
└────────────────────────────────────────────┘
```

### 握手消息类型

| 类型 | 值 | 说明 |
|------|-----|------|
| client_hello | 1 | 客户端问候 |
| server_hello | 2 | 服务器问候 |
| new_session_ticket | 4 | 新会话票证 |
| end_of_early_data | 5 | 早期数据结束 |
| encrypted_extensions | 8 | 加密扩展 |
| certificate | 11 | 证书链 |
| certificate_request | 13 | 证书请求 |
| certificate_verify | 15 | 证书验证 |
| finished | 20 | 握手完成 |
| key_update | 24 | 密钥更新 |

### 扩展类型

TLS 1.3 定义了多种扩展，用于协商额外功能：

| 扩展名称 | 值 | 用途 |
|----------|-----|------|
| server_name | 0x0000 | 服务器名称指示（SNI） |
| supported_groups | 0x000A | 支持的密钥交换组 |
| signature_algorithms | 0x000D | 支持的签名算法 |
| application_layer_protocol_negotiation | 0x0010 | 应用层协议协商（ALPN） |
| supported_versions | 0x002B | 支持的 TLS 版本 |
| key_share | 0x0033 | 密钥交换材料 |
| pre_shared_key | 0x0029 | 预共享密钥 |
| early_data | 0x002A | 早期数据指示 |
| cookie | 0x002C | HelloRetryRequest cookie |
| psk_key_exchange_modes | 0x002D | PSK 密钥交换模式 |

### 安全特性

#### 前向保密（Forward Secrecy）

TLS 1.3 强制使用临时密钥交换（Ephemeral Diffie-Hellman），确保：
- 每个会话使用独立的临时密钥对
- 即使服务器私钥泄露，也无法解密历史通信
- 前向保密由协议设计保证，不可禁用

#### 握手加密

TLS 1.3 加密了大部分握手消息：

```
未加密的消息：
- ClientHello（完整明文）
- ServerHello（完整明文）

加密的消息：
- EncryptedExtensions
- Certificate
- CertificateVerify
- Finished
- Application Data
```

这种设计防止了被动攻击者获取证书信息、ALPN 协商结果等敏感数据。

#### 降级保护

TLS 1.3 通过以下机制防止降级攻击：

1. **supported_versions 扩展**：明确协商 TLS 版本
2. **ServerHello.random 降级标记**：服务器在检测到降级时修改 random 字段
3. **签名覆盖 ServerHello.random**：CertificateVerify 签名包含 server_random

```
降级标记（嵌入在 ServerHello.random 后 8 字节）：
- TLS 1.2 降级: 44 4F 57 4E 47 52 44 01
- TLS 1.1 或更低: 44 4F 57 4E 47 52 44 00
```

### 会话恢复

TLS 1.3 支持两种会话恢复方式：

#### PSK（预共享密钥）

服务器在握手完成后发送 NewSessionTicket 消息：

```
NewSessionTicket:
┌────────────────────────────────────────────┐
│ ticket_lifetime: 4 bytes                   │
│ ticket_age_add: 4 bytes                    │
│ ticket_nonce: variable                     │
│ ticket: variable (加密的会话密钥)           │
│ extensions: variable                       │
└────────────────────────────────────────────┘
```

客户端在后续连接中使用 pre_shared_key 扩展恢复会话。

#### 会话票证流程

```
首次连接：
Client                                    Server
  |                                          |
  |<==== Application Data ==================>|
  |                                          |
  |<--------- NewSessionTicket -------------|  (发送票证)
  |                                          |

恢复连接：
Client                                    Server
  |                                          |
  |---------- ClientHello ------------------>|
  |          (pre_shared_key: 票证)           |
  |                                          |
  |<--------- ServerHello -------------------|
  |          (pre_shared_key: 选中票证)       |
  |                                          |
  |<========= Application Data ==============>|
```

## 在 Prism 中的应用

Prism 在多个模块中使用 TLS 1.3 协议，主要用于伪装层（Stealth Layer）和 Reality 协议。

### Reality 握手实现

Reality 协议利用 TLS 1.3 握手进行身份验证和密钥交换：

| 函数/类 | 功能 | 文件位置 |
|---------|------|----------|
| `parse_client_hello` | 解析 ClientHello 消息 | `stealth/reality/request.cpp` |
| `generate_server_hello` | 生成 ServerHello 消息 | `stealth/reality/response.cpp` |
| `derive_handshake_keys` | TLS 1.3 密钥派生 | `stealth/reality/keygen.cpp` |
| `encrypt_tls_record` | 加密 TLS 记录 | `stealth/reality/seal.cpp` |
| `decrypt_tls_record` | 解密 TLS 记录 | `stealth/reality/open.cpp` |

### 密钥派生实现

```cpp
// Reality 密钥派生示例（伪代码）
auto early_secret = hkdf_extract(zero, psk_or_zero);
auto derived_secret = hkdf_expand_label(early_secret, "derived", empty);
auto handshake_secret = hkdf_extract(derived_secret, ecdhe_shared_secret);
auto client_hs_key = hkdf_expand_label(handshake_secret, "c hs traffic", hash);
auto server_hs_key = hkdf_expand_label(handshake_secret, "s hs traffic", hash);
```

### 扩展解析

Prism 解析 TLS 1.3 扩展用于：

- **SNI 提取**：提取服务器名称用于路由
- **key_share 提取**：提取公钥用于密钥交换
- **supported_versions 检测**：确认 TLS 1.3 支持
- **ALPN 协商**：确定应用协议

### 协议识别

Prism 的 Recognition 模块通过 TLS ClientHello 特征识别：

```
Probe → detect() → protocol_type::tls
  │
  └→ ClientHello 分析:
      - supported_versions 包含 0x0304 → TLS 1.3
      - key_share 包含 x25519 → ECDHE 支持
      - cipher_suites 包含 0x1301 → AES-128-GCM 支持
```

## 最佳实践

### 密码套件选择

1. **优先使用 AES-GCM**：在有 AES-NI 指令集的平台上性能最佳
2. **备选 ChaCha20-Poly1305**：在 ARM 或无硬件加速的平台上更高效
3. **避免 CCM-8**：仅用于资源极度受限的设备

### 密钥交换算法选择

1. **推荐 x25519**：
   - 性能优秀（约 2 倍于 NIST P-256）
   - 常数时间实现，安全
   - 广泛部署和测试

2. **高安全场景使用 x448**：
   - 更长的密钥，更高的安全裕度
   - 性能开销较大

### 会话票证管理

1. **票证生命周期**：设置合理的 ticket_lifetime（建议 24-72 小时）
2. **票证加密**：使用强加密密钥保护票证
3. **票证轮换**：定期更换票证加密密钥
4. **限制复用**：限制单个票证的使用次数

### 安全配置

1. **禁用 0-RTT 敏感操作**：0-RTT 数据可能被重放，避免幂等操作
2. **证书验证**：始终验证服务器证书链
3. **OCSP Stapling**：启用 OCSP Stapling 减少验证延迟
4. **HSTS**：配合 HSTS 强制使用 HTTPS

### 性能优化

1. **会话复用**：启用 PSK 会话恢复减少握手开销
2. **证书优化**：使用 ECDSA 证书（比 RSA 更小更快）
3. **ALPN 协商**：提前协商应用协议避免额外 RTT
4. **连接复用**：复用 TCP 连接减少握手次数

## 常见问题

### Q1: TLS 1.3 与 TLS 1.2 可以共存吗？

是的，大多数服务器支持 TLS 1.2 和 TLS 1.3 共存。客户端在 ClientHello 中同时声明支持的版本，服务器选择最高支持的版本。如果服务器不支持 TLS 1.3，会降级到 TLS 1.2。

### Q2: 为什么 TLS 1.3 移除了 RSA 密钥交换？

RSA 密钥交换不提供前向保密。如果服务器私钥泄露，攻击者可以解密所有历史通信。TLS 1.3 强制使用 ECDHE/DHE，确保每个会话使用独立的临时密钥。

### Q3: 0-RTT 数据有什么安全风险？

0-RTT 数据存在重放攻击风险：
- 攻击者可以截获并重放 0-RTT 数据
- 服务器无法区分原始请求和重放请求
- 仅适用于幂等操作（如 GET 请求）

**缓解措施**：
- 服务器实现 0-RTT 重放检测
- 仅允许幂等操作使用 0-RTT
- 限制 0-RTT 数据的大小和使用频率

### Q4: 如何处理 TLS 1.3 的中间件兼容性？

某些网络设备（防火墙、负载均衡器）可能不兼容 TLS 1.3：

**解决方案**：
1. 配置设备支持 TLS 1.3 或 TLS 1.2 降级
2. 使用 TCP 层代理而非 TLS 终止代理
3. 部署支持 TLS 1.3 的现代中间件

### Q5: TLS 1.3 握手失败如何排查？

常见排查步骤：

1. **检查版本协商**：确认 supported_versions 扩展正确
2. **检查 key_share**：确认双方支持相同的密钥交换算法
3. **检查证书**：确认证书有效且签名算法匹配
4. **检查加密套件**：确认双方支持相同的密码套件
5. **检查网络**：确认无中间件干扰

### Q6: TLS 1.3 支持 SNI 加密吗？

TLS 1.3 本身不加密 SNI，但提供了 ESNI（Encrypted SNI）和 ECH（Encrypted Client Hello）扩展：

- **ESNI**：加密 SNI 扩展，已被 ECH 取代
- **ECH**：加密整个 ClientHello，包括 SNI

ECH 使用服务器的公钥加密 ClientHello，防止被动攻击者获取 SNI。

### Q7: 如何在 Prism 中调试 TLS 1.3 握手？

Prism 提供详细的日志记录：

```cpp
// 启用详细日志
trace::set_log_level(spdlog::level::debug);

// 关键日志点
// - ClientHello 解析
// - ServerHello 生成
// - 密钥派生
// - 记录加密/解密
```

### Q8: TLS 1.3 的 Finished 消息如何验证？

Finished 消息使用 HMAC 验证握手完整性：

```
Finished.verify_data = HMAC(finished_key, Hash(handshake_messages))
```

双方独立计算 verify_data，验证握手消息未被篡改。

## 参考资料

- [RFC 8446 - The Transport Layer Security (TLS) Protocol Version 1.3](https://tools.ietf.org/html/rfc8446)
- [RFC 8446 Appendix C - Implementation Pitfalls](https://tools.ietf.org/html/rfc8446#appendix-C)
- [TLS 1.3 - Wikimedia](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.3)
- [A Detailed Look at TLS 1.3](https://www.cloudflare.com/learning/ssl/why-use-tls-1.3/)

## 相关知识

- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 消息详解
- [[ref/protocol/tls-serverhello|TLS ServerHello]] — ServerHello 消息详解
- [[ref/protocol/tls-extensions|TLS 扩展]] — TLS 扩展机制
- [[ref/protocol/tls-sessionid|TLS SessionID]] — SessionID 字段
- [[ref/crypto/hkdf|HKDF]] — TLS 1.3 密钥派生
- [[ref/crypto/aes-gcm|AES-GCM]] — TLS 1.3 密码套件
- [[ref/crypto/x25519|X25519]] — TLS 1.3 密钥交换
- [[ref/stealth/reality|Reality 协议]] — 基于 TLS 1.3 的伪装