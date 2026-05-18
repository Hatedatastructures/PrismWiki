---
title: "TLS ServerHello 消息"
category: "protocol"
type: ref
module: ref
source: "RFC 8446 Section 4.1.3"
tags: [协议, tls, serverhello, 握手, 密钥交换, 密码套件]
created: 2026-05-15
updated: 2026-05-17
related: [tls-1-3, tls-clienthello, tls-serverrandom, tls-extensions, reality]
layer: ref
---

# TLS ServerHello 消息

**类别**: 协议

## 概述

ServerHello 是 TLS 握手协议中服务器发送给客户端的第二条消息，用于响应客户端的 ClientHello。ServerHello 消息包含了服务器选择的协议版本、密码套件、压缩方法以及必要的扩展。在 TLS 1.3 中，ServerHello 是握手过程中最后一条未加密的消息，之后的所有消息都将使用握手密钥进行加密。

ServerHello 消息在 TLS 协议栈中扮演关键的协商角色：

1. **版本确认**：明确告知客户端最终使用的 TLS 版本
2. **密码套件选择**：从客户端提供的列表中选择一个密码套件
3. **密钥交换材料**：提供服务器端的 key_share 用于密钥计算
4. **扩展响应**：回应客户端的扩展请求

在代理和伪装领域，ServerHello 消息也有重要应用：

- **Reality 协议**：服务器生成伪装的 ServerHello 响应
- **ShadowTLS 协议**：利用 ServerRandom 字段生成密钥
- **TLS 指纹**：JA3S 指纹基于 ServerHello 特征

### 消息重要性

ServerHello 消息确定了握手后续流程的所有参数：

```
ServerHello 决定：
├── selected_version → 确定 TLS 版本
├── selected_cipher_suite → 确定加密方式
├── selected_group → 确定密钥交换算法
├── server_random → 参与密钥派生
├── extensions:
│   ├── supported_versions → TLS 1.3 版本确认
│   ├── key_share → ECDHE 密钥交换完成
│   └── pre_shared_key → 会话恢复确认
└── session_id → 回显或新建
```

### TLS 1.2 与 TLS 1.3 的差异

ServerHello 消息在 TLS 1.2 和 TLS 1.3 中有重要差异：

| 特性 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| 版本协商 | 直接在 version 字段 | supported_versions 扩展 |
| 密钥交换 | 可能使用 RSA | 强制 ECDHE/DHE |
| session_id | 用于会话恢复 | 回显，主要用于兼容 |
| 后续消息 | Certificate 等明文发送 | 加密发送 |
| 扩展支持 | 可选 | 多数必需 |

### HelloRetryRequest

当服务器无法直接处理客户端的 ClientHello 时（如缺少合适的 key_share），会发送 HelloRetryRequest 消息：

```
Client                                    Server
  |                                          |
  |---------- ClientHello ------------------>|
  |                                          |
  |<---- HelloRetryRequest -----------------|
  |      (请求客户端提供特定的 key_share)     |
  |                                          |
  |---------- ClientHello (修正) ----------->|
  |                                          |
  |<--------- ServerHello -------------------|
```

HelloRetryRequest 的格式与 ServerHello 相同，但 version 字段为 0x0303（TLS 1.2 的 legacy_version），并包含特定的 cookie 扩展。

## 协议详解

### 消息结构

ServerHello 消息的完整二进制结构如下：

```
Handshake:
┌────────────────────────────────────────────┐
│ HandshakeType: 1 byte (0x02)              │
│ Length: 3 bytes                           │
│ Body: ServerHello                         │
└────────────────────────────────────────────┘

ServerHello:
┌────────────────────────────────────────────┐
│ legacy_version: 2 bytes                    │
│ random: 32 bytes                          │
│ legacy_session_id_echo: 0-32 bytes        │
│ cipher_suite: 2 bytes                     │
│ legacy_compression_method: 1 byte         │
│ extensions: variable                       │
└────────────────────────────────────────────┘
```

#### legacy_version 字段

`legacy_version` 字段长度为 2 字节，用于兼容性：

| 值 | 版本 |
|------|--------|
| 0x0303 | TLS 1.2 (TLS 1.3 也使用此值) |
| 0x0302 | TLS 1.1 |
| 0x0301 | TLS 1.0 |

**TLS 1.3 注意事项**：
- legacy_version 必须为 0x0303
- 实际版本通过 supported_versions 扩展确认
- 客户端必须检查 supported_versions 扩展确认版本

#### random 字段

`random` 字段长度为 32 字节，由服务器生成：

```
ServerHello.random:
┌────────────────────────────────────────────┐
│ Random Bytes: 32 bytes                     │
│ (TLS 1.3: 全随机，无 gmt_unix_time)        │
└────────────────────────────────────────────┘
```

**TLS 1.3 降级保护机制**：

如果服务器协商了低于 TLS 1.3 的版本（降级），ServerHello.random 的最后 8 字节会被设置为固定值：

```
降级标记（嵌入在 random[24:32]）：
- TLS 1.2 降级: 44 4F 57 4E 47 52 44 01
- TLS 1.1 或更低: 44 4F 57 4E 47 52 44 00

客户端检测：
if (random[24:32] == "DOWNGRD\x01") {
    // 服务器可能被攻击者强制降级到 TLS 1.2
    // 客户端应警告或拒绝连接
}
if (random[24:32] == "DOWNGRD\x00") {
    // 服务器可能被攻击者强制降级到 TLS 1.1 或更低
    // 客户端应拒绝连接
}
```

#### legacy_session_id_echo 字段

`legacy_session_id_echo` 字段回显客户端的 session_id：

```
legacy_session_id_echo:
┌────────────────────────────────────────────┐
│ length: 1 byte (0-32)                     │
│ session_id: 0-32 bytes                    │
└────────────────────────────────────────────┘
```

- 长度必须与 ClientHello 中的 session_id 长度相同
- 内容必须与 ClientHello 中的 session_id 相同
- TLS 1.3 中主要用于兼容性

#### cipher_suite 字段

`cipher_suite` 字段长度为 2 字节，表示服务器选择的密码套件：

```
cipher_suite:
┌────────────────────────────────────────────┐
│ Cipher Suite Value: 2 bytes               │
└────────────────────────────────────────────┘
```

TLS 1.3 密码套件：

| 值 | 名称 |
|-----|------|
| 0x1301 | TLS_AES_128_GCM_SHA256 |
| 0x1302 | TLS_AES_256_GCM_SHA384 |
| 0x1303 | TLS_CHACHA20_POLY1305_SHA256 |
| 0x1304 | TLS_AES_128_CCM_SHA256 |
| 0x1305 | TLS_AES_128_CCM_8_SHA256 |

**选择原则**：
- 必须是客户端 cipher_suites 列表中的值
- 服务器根据安全和性能偏好选择
- 优先选择前向保密的套件（TLS 1.3 全部支持）

#### legacy_compression_method 字段

`legacy_compression_method` 字段固定为 0x00：

```
legacy_compression_method:
┌────────────────────────────────────────────┐
│ Method: 1 byte (必须为 0x00)              │
└────────────────────────────────────────────┘
```

TLS 1.3 禁用压缩，此值必须为 null（0x00）。

#### extensions 字段

`extensions` 字段包含服务器选择的扩展：

```
extensions:
┌────────────────────────────────────────────┐
│ length: 2 bytes (总字节数)                 │
│ Extension_1: variable                      │
│ Extension_2: variable                      │
│ ...                                        │
└────────────────────────────────────────────┘
```

TLS 1.3 ServerHello 中允许的扩展：

| 扩展 | 值 | 说明 |
|------|-----|------|
| supported_versions | 0x002B | 确认 TLS 1.3 版本 |
| key_share | 0x0033 | 提供 ECDHE 公钥 |
| pre_shared_key | 0x0029 | 确认 PSK 选择 |
| cookie | 0x002C | HelloRetryRequest cookie |

### 关键扩展详解

#### supported_versions 扩展（ServerHello）

服务器使用 supported_versions 扩展确认最终版本：

```
supported_versions (ServerHello):
┌────────────────────────────────────────────┐
│ ExtensionType: 0x002B (2 bytes)          │
│ ExtensionData length: 1 byte              │
│ selected_version: 2 bytes (0x0304)       │
└────────────────────────────────────────────┘
```

**重要**：客户端必须检查此扩展，而不是依赖 legacy_version 字段。

#### key_share 扩展（ServerHello）

服务器提供密钥交换公钥：

```
key_share (ServerHello):
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0033 (2 bytes)          │
│ ExtensionData length: 2 bytes            │
│ server_share:                             │
│   named_group: 2 bytes                    │
│   key_exchange length: 2 bytes           │
│   key_exchange: variable                  │
└────────────────────────────────────────────┘
```

**named_group 必须匹配**：
- 如果 ClientHello 的 key_share 中有服务器支持的组，直接使用该组
- 如果 ClientHello 的 key_share 中没有服务器支持的组，发送 HelloRetryRequest

#### pre_shared_key 扩展（ServerHello）

确认 PSK 选择（会话恢复）：

```
pre_shared_key (ServerHello):
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0029 (2 bytes)          │
│ ExtensionData length: 2 bytes            │
│ selected_identity: 2 bytes               │
└────────────────────────────────────────────┘
```

`selected_identity` 表示服务器选择了客户端 pre_shared_key 扩展中第几个 PSK（从 0 开始计数）。

#### cookie 扩展（HelloRetryRequest）

HelloRetryRequest 使用 cookie 扩展：

```
cookie (HelloRetryRequest):
┌────────────────────────────────────────────┐
│ ExtensionType: 0x002C (2 bytes)          │
│ ExtensionData length: 2 bytes            │
│ cookie: variable                          │
└────────────────────────────────────────────┘
```

客户端在重发 ClientHello 时必须回显此 cookie。

### TLS 记录层封装

ServerHello 消息被封装在 TLS 记录层中：

```
TLS Record:
┌────────────────────────────────────────────┐
│ ContentType: 0x16 (Handshake) - 1 byte    │
│ Version: 0x0303 (TLS 1.2) - 2 bytes       │
│ Length: N - 2 bytes                        │
├────────────────────────────────────────────┤
│ Handshake:                                │
│   Type: 0x02 (ServerHello) - 1 byte       │
│   Length: N-4 - 3 bytes                   │
│   Body: ServerHello content               │
└────────────────────────────────────────────┘
```

### 密钥计算流程

ServerHello 发送后，双方可以计算握手密钥：

```
密钥计算流程（TLS 1.3）：

1. ECDHE 密钥交换：
   client_private = 随机生成
   client_public = X25519(client_private, base_point)
   
   server_private = 随机生成
   server_public = X25519(server_private, base_point)
   
   shared_secret = X25519(client_private, server_public)
                 = X25519(server_private, client_public)

2. HKDF 密钥派生：
   early_secret = HKDF-Extract(0, 0^hash_len)
   derived_secret = HKDF-Expand-Label(early_secret, "derived", "")
   handshake_secret = HKDF-Extract(derived_secret, shared_secret)
   
   client_handshake_traffic_secret = 
       HKDF-Expand-Label(handshake_secret, "c hs traffic", transcript_hash)
   server_handshake_traffic_secret = 
       HKDF-Expand-Label(handshake_secret, "s hs traffic", transcript_hash)

3. 握手密钥：
   client_hs_key = Derive-Secret(client_handshake_traffic_secret, "key")
   client_hs_iv = Derive-Secret(client_handshake_traffic_secret, "iv")
   server_hs_key = Derive-Secret(server_handshake_traffic_secret, "key")
   server_hs_iv = Derive-Secret(server_handshake_traffic_secret, "iv")
```

### HelloRetryRequest 详细流程

HelloRetryRequest 在以下情况触发：

1. 客户端 key_share 中没有服务器支持的组
2. 服务器需要额外的信息（如 cookie）

```
HelloRetryRequest 流程：

Client                                    Server
  |                                          |
  |---------- ClientHello ------------------>|
  |   key_share: {secp256r1, secp384r1}     |
  |                                          |
  |                                          | ← 服务器只支持 x25519
  |                                          |
  |<---- HelloRetryRequest -----------------|
  |   supported_versions: 0x0304            |
  |   key_share: {named_group: x25519}      |
  |   cookie: [随机 cookie]                  |
  |                                          |
  |---------- ClientHello (重发) ----------->|
  |   key_share: {x25519, [x25519公钥]}     |
  |   cookie: [回显 cookie]                  |
  |                                          |
  |<--------- ServerHello -------------------|
  |   key_share: {x25519, [服务器公钥]}      |
  |                                          |
  |<========= Encrypted Extensions ==========|
  ...

HelloRetryRequest 格式：
┌────────────────────────────────────────────┐
│ HandshakeType: 0x02 (ServerHello)         │
│ legacy_version: 0x0303                    │
│ random: [HelloRetryRequest 特定值]        │
│   random = SHA-256("HelloRetryRequest")  │
│ session_id_echo: 回显                     │
│ cipher_suite: 选择的套件                   │
│ compression_method: 0x00                   │
│ extensions:                               │
│   - supported_versions (0x0304)           │
│   - key_share (仅 named_group)            │
│   - cookie                                │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### ServerHello 生成

Prism 的 Reality 模块生成 ServerHello 用于伪装响应：

```cpp
// 文件: src/prism/stealth/reality/response.cpp
auto generate_server_hello(const client_hello_features& client_features,
                           const reality_config& config)
    -> std::vector<std::byte>
{
    std::vector<std::byte> server_hello;
    
    // legacy_version: 0x0303
    append_u16(server_hello, 0x0303);
    
    // random: 32 字节（包含 Reality 特征或随机）
    append_random(server_hello, 32);
    
    // session_id_echo: 回显客户端 session_id
    append_session_id(server_hello, client_features.session_id);
    
    // cipher_suite: 选择第一个支持的套件
    append_u16(server_hello, select_cipher_suite(client_features));
    
    // compression_method: 0x00
    append_u8(server_hello, 0x00);
    
    // extensions
    append_extensions(server_hello, {
        supported_versions(0x0304),
        key_share(config.key_share_group, server_public_key)
    });
    
    return server_hello;
}
```

### 密钥派生

Reality 使用 ServerHello.random 参与密钥派生：

```cpp
// 文件: src/prism/stealth/reality/keygen.cpp
auto derive_handshake_keys(const std::vector<std::byte>& shared_secret,
                           const std::vector<std::byte>& client_hello_hash,
                           const std::vector<std::byte>& server_hello_hash)
    -> handshake_keys
{
    // Transcript hash 包含 ClientHello 和 ServerHello
    auto transcript_hash = sha256(client_hello + server_hello);
    
    // 派生握手密钥
    auto handshake_secret = hkdf_extract(derived_secret, shared_secret);
    
    return handshake_keys{
        .client_key = hkdf_expand_label(handshake_secret, "c hs traffic", transcript_hash),
        .server_key = hkdf_expand_label(handshake_secret, "s hs traffic", transcript_hash)
    };
}
```

### ShadowTLS ServerRandom 使用

ShadowTLS 使用 ServerRandom 生成写入密钥：

```cpp
// 文件: src/prism/stealth/shadowtls/auth.cpp
auto compute_write_key(std::string_view password,
                       std::span<const std::byte> server_random)
    -> std::vector<std::byte>
{
    // WriteKey = SHA256(password + serverRandom)
    std::vector<std::byte> buffer;
    buffer.append(password.begin(), password.end());
    buffer.append(server_random.begin(), server_random.end());
    
    return sha256(buffer);
}
```

### ServerHello 解析

Prism 解析 ServerHello 用于验证握手：

```cpp
// 文件: src/prism/stealth/reality/request.cpp
auto parse_server_hello(std::span<const std::byte> data)
    -> outcome::result<server_hello_features>
{
    // 解析 legacy_version
    auto legacy_version = read_u16(data);
    
    // 解析 random
    auto random = data.subspan(2, 32);
    
    // 检查降级标记
    if (check_downgrade_marker(random)) {
        // 检测到降级攻击
    }
    
    // 解析 session_id_echo
    auto session_id_echo = parse_session_id(data);
    
    // 解析 cipher_suite
    auto cipher_suite = read_u16(data);
    
    // 解析 extensions
    auto extensions = parse_extensions(data);
    
    return server_hello_features{
        .version = extract_version_from_extensions(extensions),
        .random = random,
        .cipher_suite = cipher_suite,
        .key_share = extract_key_share(extensions)
    };
}
```

## 最佳实践

### ServerHello 构造

1. **正确的 legacy_version**：TLS 1.3 必须使用 0x0303
2. **强随机数**：使用加密安全的随机数生成 ServerRandom
3. **版本确认**：必须包含 supported_versions 扩展
4. **key_share 匹配**：选择客户端支持的组
5. **降级保护**：在降级时正确设置 random 标记

### 密码套件选择策略

1. **优先 AES-GCM**：在有 AES-NI 的平台上性能最佳
2. **备选 ChaCha20**：在无硬件加速的平台上更高效
3. **考虑客户端偏好**：尊重客户端 cipher_suites 的顺序
4. **安全优先**：避免选择弱密码套件

### HelloRetryRequest 处理

1. **快速响应**：尽快发送 HelloRetryRequest
2. **Cookie 安全**：使用加密安全的随机 cookie
3. **状态保存**：保存 ClientHello hash 用于后续 transcript 计算
4. **限制重试**：限制 HelloRetryRequest 的次数

### 扩展顺序

ServerHello 扩展顺序建议：
- supported_versions 最先（版本确认）
- key_share 其次（密钥交换）
- pre_shared_key 最后（会话恢复）

## 常见问题

### Q1: ServerHello 之后为什么消息都加密了？

ServerHello 发送后，双方已经计算出握手密钥（handshake traffic secret），可以使用这些密钥加密后续消息。这保护了 Certificate、CertificateVerify 等敏感信息不被被动攻击者获取。

### Q2: 如何区分 ServerHello 和 HelloRetryRequest？

HelloRetryRequest 的格式与 ServerHello 相同，但有以下特征：

1. **ServerHello.random**：固定为 SHA-256("HelloRetryRequest") 的值
2. **key_share 扩展**：只包含 named_group，不包含 key_exchange
3. **cookie 扩展**：通常包含 cookie

```
HelloRetryRequest.random 固定值（32 字节）：
CF 21 AD 74 E5 9A 61 11
BE 1D 8C 02 1E 65 B8 91
C2 A2 11 16 7A BB 8C 5E
07 9E 09 E2 C8 A8 33 9C
```

### Q3: ServerHello 降级标记如何工作？

降级保护机制防止攻击者强制降级 TLS 版本：

1. 客户端发送 ClientHello 支持 TLS 1.3
2. 服务器协商低于 TLS 1.3 的版本（被攻击者干扰）
3. 服务器在 ServerHello.random[24:32] 设置降级标记
4. 客户端检测标记，警告或拒绝连接

**注意**：如果服务器确实只支持 TLS 1.2，则不设置降级标记。

### Q4: Session ID Echo 必须完全匹配吗？

是的，ServerHello 中的 session_id_echo 必须：
- 长度与 ClientHello 的 session_id 相同
- 内容与 ClientHello 的 session_id 相同

如果不匹配，客户端必须中止握手。

### Q5: 为什么有些 ServerHello 没有 key_share？

以下情况 ServerHello 可能没有 key_share：

1. **使用 PSK 密钥交换**：pre_shared_key 扩展已提供密钥
2. **HelloRetryRequest**：只指定 named_group，等待客户端重发
3. **TLS 1.2**：使用 RSA 或 DH 密钥交换

### Q6: Prism 如何处理 HelloRetryRequest？

Prism 的处理流程：

1. 解析 HelloRetryRequest，提取 named_group 和 cookie
2. 生成新的 ClientHello，包含正确的 key_share 和 cookie
3. 重新计算 transcript hash
4. 发送修正的 ClientHello，继续握手

### Q7: ServerHello 最小大小是多少？

不含扩展的最小 ServerHello 约 38 字节：
- legacy_version: 2 bytes
- random: 32 bytes
- session_id_echo length: 1 byte (0)
- cipher_suite: 2 bytes
- compression_method: 1 byte

TLS 1.3 要求包含 supported_versions 扩展，典型大小为 40-100 字节。

### Q8: 如何验证 ServerHello 的有效性？

客户端验证 ServerHello 的检查项：

1. **cipher_suite 检查**：必须是 ClientHello cipher_suites 中的值
2. **版本检查**：必须通过 supported_versions 确认
3. **key_share 检查**：必须使用客户端支持的 named_group
4. **降级检查**：检查 random 是否包含降级标记
5. **session_id 检查**：必须与 ClientHello 的 session_id 匹配
6. **compression 检查**：必须为 0x00

## 参考资料

- [RFC 8446 Section 4.1.3 - ServerHello](https://tools.ietf.org/html/rfc8446#section-4.1.3)
- [RFC 8446 Section 4.1.4 - HelloRetryRequest](https://tools.ietf.org/html/rfc8446#section-4.1.4)
- [RFC 8446 Section 4.2 - Extensions](https://tools.ietf.org/html/rfc8446#section-4.2)
- [TLS Extensions IANA Registry](https://www.iana.org/assignments/tls-extensiontype-values/)

## 相关知识

- [[ref/protocol/tls-1-3|TLS 1.3]] — TLS 1.3 协议详解
- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 消息详解
- [[ref/protocol/tls-serverrandom|TLS ServerRandom]] — ServerRandom 字段详解
- [[ref/protocol/tls-extensions|TLS 扩展]] — 扩展机制详解
- [[ref/crypto/hkdf|HKDF]] — TLS 1.3 密钥派生
- [[ref/crypto/x25519|X25519]] — TLS 1.3 密钥交换
- [[ref/stealth/reality|Reality 协议]] — Reality 伪装响应