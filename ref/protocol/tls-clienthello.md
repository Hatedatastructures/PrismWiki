---
title: "TLS ClientHello 消息"
category: "protocol"
type: ref
module: ref
source: "RFC 8446 Section 4.1.2"
tags: [协议, tls, clienthello, 握手, 伪装, 指纹]
created: 2026-05-15
updated: 2026-05-17
related: [tls-1-3, tls-serverhello, tls-extensions, tls-sessionid, reality, shadowtls]
---

# TLS ClientHello 消息

**类别**: 协议

## 概述

ClientHello 是 TLS 握手协议中客户端发送的第一条消息，用于发起 TLS 连接建立。这条消息包含了客户端支持的所有协议版本、密码套件、压缩方法和扩展。服务器收到 ClientHello 后，根据其中携带的信息选择适当的参数，然后回复 ServerHello 消息。

ClientHello 消息在 TLS 协议栈中具有至关重要的地位：

1. **协议协商**：声明客户端支持的 TLS 版本和功能
2. **密码协商**：列出客户端支持的密码套件供服务器选择
3. **功能扩展**：通过扩展机制传递额外参数（如 SNI、ALPN 等）
4. **安全基础**：包含密钥交换所需的随机数和公钥材料

在代理和伪装领域，ClientHello 消息尤为重要：

- **Reality 协议**：利用 SessionID 字段嵌入身份标记
- **ShadowTLS 协议**：在 SessionID 字段嵌入 HMAC 标签
- **TLS 指纹**：JA3/JA4 等指纹技术基于 ClientHello 特征

### 历史演变

ClientHello 消息格式经历了多次演变：

- **SSL 2.0**：使用不同的消息格式，现已废弃
- **SSL 3.0 / TLS 1.0-1.2**：基本结构相同，扩展字段为可选
- **TLS 1.3**：扩展字段变为必需，增加 key_share 等新扩展

### 消息重要性

ClientHello 消息是 TLS 握手的起点，决定了后续握手流程：

```
ClientHello 内容影响：
├── protocol_version → 选择 TLS 版本
├── cipher_suites → 选择密码套件
├── extensions:
│   ├── supported_versions → TLS 1.3 版本协商
│   ├── key_share → ECDHE 密钥交换
│   ├── server_name → 服务器证书选择
│   ├── alpn → 应用协议协商
│   ├── signature_algorithms → 签名算法选择
│   └── pre_shared_key → 会话恢复
└── SessionID → 会话恢复（TLS 1.2）或伪装标记
```

## 协议详解

### 消息结构

ClientHello 消息的完整二进制结构如下：

```
Handshake:
┌────────────────────────────────────────────┐
│ HandshakeType: 1 byte (0x01)              │
│ Length: 3 bytes                           │
│ Body: ClientHello                         │
└────────────────────────────────────────────┘

ClientHello:
┌────────────────────────────────────────────┐
│ legacy_version: 2 bytes                    │
│ random: 32 bytes                          │
│ legacy_session_id: 0-32 bytes              │
│ cipher_suites: 2 + N*2 bytes              │
│ legacy_compression_methods: 1 + N bytes   │
│ extensions: variable                       │
└────────────────────────────────────────────┘
```

#### legacy_version 字段

`legacy_version` 字段长度为 2 字节，表示客户端支持的 TLS 版本：

| 值 | 版本 |
|------|--------|
| 0x0300 | SSL 3.0（已废弃） |
| 0x0301 | TLS 1.0 |
| 0x0302 | TLS 1.1 |
| 0x0303 | TLS 1.2 |

**注意**：TLS 1.3 客户端也使用 0x0303 作为 `legacy_version`，实际版本通过 `supported_versions` 扩展协商。

#### random 字段

`random` 字段长度为 32 字节，由客户端生成的随机数：

```
ClientHello.random:
┌────────────────────────────────────────────┐
│ gmt_unix_time: 4 bytes (遗留，现可为随机)   │
│ random_bytes: 28 bytes                     │
└────────────────────────────────────────────┘
```

- 用于密钥派生
- 防止重放攻击
- TLS 1.3 中与 ServerHello.random 一起参与密钥计算

#### legacy_session_id 字段

`legacy_session_id` 字段长度为 0-32 字节：

```
legacy_session_id:
┌────────────────────────────────────────────┐
│ length: 1 byte (0-32)                     │
│ session_id: 0-32 bytes                    │
└────────────────────────────────────────────┘
```

在 TLS 1.2 中用于会话恢复。TLS 1.3 中保留为兼容性，但有特殊用途：

- **Reality 协议**：前 3 字节作为独占标记
- **ShadowTLS 协议**：后 4 字节嵌入 HMAC 标签

#### cipher_suites 字段

`cipher_suites` 字段列出客户端支持的密码套件：

```
cipher_suites:
┌────────────────────────────────────────────┐
│ length: 2 bytes (字节数)                   │
│ cipher_suite_1: 2 bytes                   │
│ cipher_suite_2: 2 bytes                   │
│ ...                                        │
│ cipher_suite_N: 2 bytes                   │
└────────────────────────────────────────────┘
```

TLS 1.3 定义的密码套件：

| 值 | 名称 |
|-----|------|
| 0x1301 | TLS_AES_128_GCM_SHA256 |
| 0x1302 | TLS_AES_256_GCM_SHA384 |
| 0x1303 | TLS_CHACHA20_POLY1305_SHA256 |
| 0x1304 | TLS_AES_128_CCM_SHA256 |
| 0x1305 | TLS_AES_128_CCM_8_SHA256 |

**顺序重要性**：客户端应按优先级从高到低排列密码套件。

#### legacy_compression_methods 字段

`legacy_compression_methods` 字段在 TLS 1.3 中固定为 `{0x01, 0x00}`：

```
legacy_compression_methods:
┌────────────────────────────────────────────┐
│ length: 1 byte (必须为 0x01)              │
│ method: 1 byte (必须为 0x00 = null)       │
└────────────────────────────────────────────┘
```

TLS 1.3 禁用了压缩功能，此字段仅用于兼容性。

#### extensions 字段

`extensions` 字段包含零个或多个扩展：

```
extensions:
┌────────────────────────────────────────────┐
│ length: 2 bytes (总字节数)                 │
│ Extension_1: variable                      │
│ Extension_2: variable                      │
│ ...                                        │
│ Extension_N: variable                      │
└────────────────────────────────────────────┘

Extension:
┌────────────────────────────────────────────┐
│ ExtensionType: 2 bytes                    │
│ ExtensionData length: 2 bytes             │
│ ExtensionData: variable                   │
└────────────────────────────────────────────┘
```

### 关键扩展详解

#### supported_versions 扩展（0x002B）

用于协商 TLS 版本，TLS 1.3 必需：

```
supported_versions:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x002B (2 bytes)          │
│ ExtensionData length: 2 bytes            │
│ supported_version_list:                   │
│   list_length: 1 byte                     │
│   version_1: 2 bytes (e.g., 0x0304)      │
│   version_2: 2 bytes                      │
│   ...                                     │
└────────────────────────────────────────────┘
```

TLS 1.3 版本码：`0x0304`

#### key_share 扩展（0x0033）

携带密钥交换的公钥材料：

```
key_share:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0033 (2 bytes)          │
│ ExtensionData length: 2 bytes            │
│ client_shares:                            │
│   KeyShareEntry_1:                        │
│     named_group: 2 bytes                  │
│     key_exchange length: 2 bytes         │
│     key_exchange: variable                │
│   KeyShareEntry_2: ...                    │
└────────────────────────────────────────────┘
```

常用 named_group 值：

| 值 | 名称 |
|------|--------|
| 0x0017 | secp256r1 (P-256) |
| 0x0018 | secp384r1 (P-384) |
| 0x0019 | secp521r1 (P-521) |
| 0x001D | x25519 |
| 0x001E | x448 |

对于 x25519，key_exchange 为 32 字节的公钥。

#### server_name 扩展（0x0000）

服务器名称指示（SNI），用于虚拟主机：

```
server_name:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0000 (2 bytes)           │
│ ExtensionData length: 2 bytes             │
│ ServerNameList:                           │
│   list_length: 2 bytes                    │
│   ServerName:                             │
│     name_type: 1 byte (0x00 = hostname)   │
│     name_length: 2 bytes                  │
│     name: variable (主机名)                │
└────────────────────────────────────────────┘
```

#### signature_algorithms 扩展（0x000D）

声明支持的签名算法：

```
signature_algorithms:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x000D (2 bytes)          │
│ ExtensionData length: 2 bytes            │
│ supported_signature_algorithms:           │
│   list_length: 2 bytes                    │
│   SignatureScheme_1: 2 bytes             │
│   SignatureScheme_2: 2 bytes             │
│   ...                                     │
└────────────────────────────────────────────┘
```

常用签名算法：

| 值 | 名称 |
|------|--------|
| 0x0403 | ecdsa_secp256r1_sha256 |
| 0x0503 | ecdsa_secp384r1_sha384 |
| 0x0603 | ecdsa_secp521r1_sha512 |
| 0x0804 | rsa_pss_rsae_sha256 |
| 0x0805 | rsa_pss_rsae_sha384 |
| 0x0806 | rsa_pss_rsae_sha512 |
| 0x0401 | rsa_pkcs1_sha256 |
| 0x0501 | rsa_pkcs1_sha384 |
| 0x0601 | rsa_pkcs1_sha512 |

#### application_layer_protocol_negotiation 扩展（0x0010）

应用层协议协商（ALPN）：

```
application_layer_protocol_negotiation:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0010 (2 bytes)          │
│ ExtensionData length: 2 bytes            │
│ ALPNProtocolList:                         │
│   list_length: 2 bytes                    │
│   Protocol_1:                              │
│     length: 1 byte                        │
│     name: variable (e.g., "h2")           │
│   Protocol_2: ...                         │
└────────────────────────────────────────────┘
```

常用协议标识：

- `h2` - HTTP/2
- `http/1.1` - HTTP/1.1
- `h3` - HTTP/3 (QUIC)

#### supported_groups 扩展（0x000A）

声明支持的密钥交换组：

```
supported_groups:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x000A (2 bytes)          │
│ ExtensionData length: 2 bytes            │
│ named_group_list:                         │
│   list_length: 2 bytes                    │
│   named_group_1: 2 bytes                  │
│   named_group_2: 2 bytes                  │
│   ...                                     │
└────────────────────────────────────────────┘
```

此扩展与 key_share 扩展配合使用，声明支持的组并可能提供对应的公钥。

### TLS 记录层封装

ClientHello 消息被封装在 TLS 记录层中：

```
TLS Record:
┌────────────────────────────────────────────┐
│ ContentType: 0x16 (Handshake) - 1 byte    │
│ Version: 0x0301 (TLS 1.0) - 2 bytes       │
│ Length: N - 2 bytes                        │
├────────────────────────────────────────────┤
│ Handshake:                                │
│   Type: 0x01 (ClientHello) - 1 byte       │
│   Length: N-4 - 3 bytes                   │
│   Body: ClientHello content               │
└────────────────────────────────────────────┘
```

### 完整流程示例

```
Client                                        Server
  |                                              |
  |-------- TLS Record Layer ------------------>|
  |  Content Type: 0x16 (Handshake)            |
  |  Version: 0x0301                           |
  |  Length: 512                               |
  |                                            |
  |  Handshake Header:                         |
  |    Type: 0x01 (ClientHello)                |
  |    Length: 508                             |
  |                                            |
  |  ClientHello:                              |
  |    legacy_version: 0x0303                  |
  |    random: [32 bytes]                      |
  |    session_id: [32 bytes]                  |
  |    cipher_suites: [16 suites]              |
  |    compression: [0x01, 0x00]               |
  |    extensions:                             |
  |      - supported_versions (0x002B)         |
  |      - key_share (0x0033)                  |
  |      - server_name (0x0000)                |
  |      - signature_algorithms (0x000D)       |
  |      - supported_groups (0x000A)            |
  |      - alpn (0x0010)                       |
  |                                            |
  |<------- ServerHello -----------------------|
```

### SessionID 字段的伪装利用

Reality 和 ShadowTLS 协议利用 SessionID 字段进行身份标记：

#### Reality 协议

Reality 在 SessionID 前 3 字节嵌入独占标记：

```
SessionID (Reality 客户端):
┌────────────────────────────────────────────┐
│ [0x01][0x08][0x02] - Reality 标记 (3B)   │
│ 随机数据 (29 bytes)                         │
└────────────────────────────────────────────┘

检测逻辑：
if (session_id[0:3] == [0x01, 0x08, 0x02]) {
    // 可能是 Reality 客户端
}
```

#### ShadowTLS 协议

ShadowTLS v3 在 SessionID 后 4 字节嵌入 HMAC 标签：

```
SessionID (ShadowTLS v3 客户端):
┌────────────────────────────────────────────┐
│ 随机数据 (28 bytes)                         │
│ HMAC-SHA1 标签 (4 bytes)                    │
└────────────────────────────────────────────┘

HMAC 计算方式：
HMAC = HMAC-SHA1(password,
                 ClientHello[10:28] +
                 0x00000000 +
                 ClientHello[32:])[:4]
```

## 在 Prism 中的应用

### ClientHello 解析

Prism 的 Reality 模块解析 ClientHello 用于提取关键信息：

```cpp
// 文件: src/prism/stealth/reality/request.cpp
auto parse_client_hello(std::span<const std::byte> data)
    -> outcome::result<client_hello_features>
{
    // 解析 legacy_version
    // 解析 random
    // 解析 session_id
    // 解析 cipher_suites
    // 解析 extensions
    
    return client_hello_features{
        .version = version,
        .random = random,
        .session_id = session_id,
        .cipher_suites = std::move(cipher_suites),
        .sni = sni,
        .key_shares = std::move(key_shares),
        .supported_versions = std::move(supported_versions),
        .signature_algorithms = std::move(signature_algorithms)
    };
}
```

### 身份验证

ShadowTLS 模块使用 SessionID HMAC 验证客户端身份：

```cpp
// 文件: src/prism/stealth/shadowtls/auth.cpp
auto verify_client_hello(const client_hello_features& features,
                         std::string_view password)
    -> bool
{
    // 提取 SessionID 后 4 字节
    auto tag = features.session_id.subspan<28, 4>();
    
    // 计算期望的 HMAC
    auto expected = compute_hmac(password, features);
    
    // 常量时间比较
    return constant_time_compare(tag, expected);
}
```

### 协议检测

Recognition 模块通过 ClientHello 特征检测协议类型：

```cpp
// 文件: src/prism/recognition/probe/analyzer.cpp
auto analyze(const client_hello_features& features)
    -> analysis_result
{
    // Tier 0: 快速检测
    // - Reality: session_id[0:3] == [0x01, 0x08, 0x02]
    // - ShadowTLS: HMAC 验证
    
    // Tier 1: 特征分析
    // - cipher_suites 顺序
    // - extensions 顺序
    // - SNI 匹配
    
    return analysis_result{
        .candidates = std::move(candidates),
        .confidence = confidence
    };
}
```

### 提取信息

Prism 从 ClientHello 中提取的关键信息：

| 信息 | 用途 | 扩展 |
|------|------|------|
| SNI | 路由、证书选择 | server_name (0x0000) |
| 支持的版本 | 版本协商 | supported_versions (0x002B) |
| 密钥交换组 | ECDHE 计算 | key_share (0x0033) |
| 签名算法 | 证书验证 | signature_algorithms (0x000D) |
| ALPN | 应用协议选择 | alpn (0x0010) |
| SessionID 标记 | 协议检测 | - |

## 最佳实践

### 构造合规的 ClientHello

1. **正确的版本号**：legacy_version 设为 0x0303，supported_versions 包含 0x0304
2. **强随机数**：使用加密安全的随机数生成器
3. **合理的密码套件顺序**：按安全性和性能排列
4. **必需扩展**：TLS 1.3 必须包含 supported_versions 和 signature_algorithms
5. **压缩方法**：固定为 null

### 安全考虑

1. **随机数生成**：使用加密安全的 PRNG（如 OpenSSL RAND_bytes）
2. **密码套件选择**：优先选择前向保密的套件
3. **扩展顺序**：某些实现可能根据扩展顺序进行指纹识别
4. **SNI 隐私**：考虑使用 ECH 保护 SNI

### 性能优化

1. **减少密码套件数量**：只发送客户端实际支持的套件
2. **优化 key_share**：只发送最可能被接受的公钥
3. **会话复用**：使用 PSK 扩展减少握手时间

## 常见问题

### Q1: 为什么 TLS 1.3 的 legacy_version 是 0x0303？

TLS 1.3 为了兼容中间件，使用 TLS 1.2 的版本号 0x0303 作为 legacy_version，实际版本通过 supported_versions 扩展协商。这样可以避免某些不识别 0x0304 版本号的中间件阻断连接。

### Q2: ClientHello 的最小大小是多少？

不含扩展的最小 ClientHello 约 40 字节：
- legacy_version: 2 bytes
- random: 32 bytes
- session_id length: 1 byte (0)
- cipher_suites length: 2 bytes + 至少 2 bytes (1 个套件)
- compression methods length: 1 byte + 1 byte (null)

实际上，TLS 1.3 要求包含必需扩展，典型大小为 200-500 字节。

### Q3: 如何防止 TLS 指纹识别？

TLS 指纹（如 JA3/JA4）基于 ClientHello 特征：

- **JA3**：MD5(version + cipher_suites + extensions + elliptic_curves + elliptic_curve_point_formats)
- **JA4**：改进的指纹算法，考虑更多因素

**缓解措施**：
1. 使用常见客户端的指纹特征
2. 随机化密码套件和扩展顺序
3. 使用 ECH 加密 ClientHello

### Q4: 为什么 key_share 可能导致握手失败？

如果客户端在 key_share 中发送了服务器不支持的组，服务器会发送 HelloRetryRequest 要求客户端重新发送支持的 key_share。这会增加一个 RTT。

**最佳实践**：客户端应在 supported_groups 中声明所有支持的组，并在 key_share 中包含最可能被接受的组（如 x25519）。

### Q5: SessionID 最大长度是多少？

SessionID 最大长度为 32 字节。TLS 1.3 中 SessionID 字段保留用于兼容性，实际会话恢复使用 PSK 机制。

### Q6: 如何解析 ClientHello 消息？

解析 ClientHello 时需要注意：
1. 检查消息长度是否合理
2. 处理变长字段时检查边界
3. 扩展可能有重复，需要正确处理
4. 忽略不认识的扩展（向前兼容）

### Q7: Prism 如何处理格式错误的 ClientHello？

Prism 的处理策略：
1. 长度不足：返回解析错误，断开连接
2. 未知扩展：忽略，继续解析
3. 重复扩展：使用最后一个出现的值
4. 无效版本：降级或拒绝连接

### Q8: 为什么需要 ALPN 扩展？

ALPN（Application-Layer Protocol Negotiation）用于协商应用层协议：

- HTTP/2 需要 ALPN 协商（协议标识 `h2`）
- 某些服务器可能拒绝没有 ALPN 的连接
- 代理服务器可能根据 ALPN 进行路由

## 参考资料

- [RFC 8446 Section 4.1.2 - ClientHello](https://tools.ietf.org/html/rfc8446#section-4.1.2)
- [RFC 8446 Section 4.2 - Extensions](https://tools.ietf.org/html/rfc8446#section-4.2)
- [TLS Extensions IANA Registry](https://www.iana.org/assignments/tls-extensiontype-values/)
- [JA3: SSL/TLS Client Fingerprinting](https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967)

## 相关知识

- [[ref/protocol/tls-1-3|TLS 1.3]] — TLS 1.3 协议详解
- [[ref/protocol/tls-serverhello|TLS ServerHello]] — ServerHello 消息
- [[ref/protocol/tls-extensions|TLS 扩展]] — 扩展机制详解
- [[ref/protocol/tls-sessionid|TLS SessionID]] — SessionID 字段详解
- [[ref/stealth/reality|Reality 协议]] — Reality 身份检测
- [[ref/stealth/shadowtls|ShadowTLS 协议]] — ShadowTLS HMAC 认证
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — JA3/JA4 指纹