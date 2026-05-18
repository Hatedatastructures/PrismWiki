---
title: "TLS 扩展机制"
category: "protocol"
type: ref
module: ref
source: "RFC 8446 Section 4.2"
tags: [协议, tls, extensions, 扩展, sni, alpn, key-share, ech]
created: 2026-05-15
updated: 2026-05-17
related: [tls-clienthello, tls-serverhello, tls-1-3, reality, ech]
layer: ref
---

# TLS 扩展机制

**类别**: 协议

## 概述

TLS 扩展是 TLS 协议中用于协商额外功能的机制，允许客户端和服务器在不修改核心协议的情况下添加新特性。扩展机制最初在 RFC 3546 (TLS Extensions) 中定义，后来被整合到 TLS 1.2 (RFC 5246) 和 TLS 1.3 (RFC 8446) 中。TLS 1.3 大幅扩展了扩展机制的使用，许多原本可选的扩展变成了必需。

TLS 扩展在 ClientHello 和 ServerHello 消息中携带，采用统一的 TLV (Type-Length-Value) 格式。每个扩展由 2 字节的类型码、2 字节的长度和变长的数据组成。客户端在 ClientHello 中声明支持的扩展，服务器在 ServerHello 中选择性地响应。

TLS 扩展在代理和伪装领域扮演着重要角色：

- **SNI (server_name)**：服务器名称指示，用于虚拟主机路由和证书选择
- **ALPN**：应用层协议协商，用于 HTTP/2 等协议协商
- **key_share**：密钥交换材料，用于 TLS 1.3 ECDHE
- **ECH**：加密 ClientHello，保护 SNI 等敏感信息

### 扩展的重要性

TLS 扩展机制的重要性体现在以下几个方面：

1. **协议演进**：在不修改核心协议的情况下添加新功能
2. **功能协商**：客户端和服务器协商支持的特性
3. **性能优化**：如会话恢复、OCSP stapling 等
4. **安全增强**：如 supported_versions、signature_algorithms 等
5. **隐私保护**：如 ECH 加密 ClientHello

### TLS 1.2 与 TLS 1.3 的扩展差异

TLS 1.3 对扩展机制进行了重大调整：

| 特性 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| 扩展位置 | ClientHello/ServerHello | ClientHello/ServerHello + EncryptedExtensions |
| 必需扩展 | 无 | supported_versions, signature_algorithms |
| 新扩展 | 较少 | key_share, pre_shared_key, early_data 等 |
| 加密扩展 | 不支持 | EncryptedExtensions 消息 |
| ECH 支持 | 不支持 | 支持 |

### 扩展类型分类

TLS 扩展可以按照功能分类：

```
TLS 扩展分类：
├── 协商类扩展
│   ├── supported_versions - TLS 版本协商
│   ├── supported_groups - 密钥交换组协商
│   ├── signature_algorithms - 签名算法协商
│   ├── alpn - 应用协议协商
│   └── compress_certificate - 证书压缩协商
│
├── 密钥交换类扩展
│   ├── key_share - ECDHE 公钥交换
│   ├── pre_shared_key - PSK 会话恢复
│   ├── psk_key_exchange_modes - PSK 模式协商
│   └── early_data - 0-RTT 数据
│
├── 路由类扩展
│   ├── server_name (SNI) - 服务器名称指示
│   └── application_layer_protocol_negotiation (ALPN)
│
├── 安全类扩展
│   ├── encrypted_client_hello (ECH) - 加密 ClientHello
│   ├── record_size_limit - 记录大小限制
│   └── cookie - HelloRetryRequest cookie
│
└── 优化类扩展
    ├── status_request (OCSP) - OCSP Stapling
    ├── max_fragment_length - 最大片段长度
    └── session_ticket - 会话票证
```

## 协议详解

### 扩展格式

TLS 扩展采用统一的 TLV 格式：

```
Extension:
┌────────────────────────────────────────────┐
│ ExtensionType: 2 bytes                    │
│ ExtensionData Length: 2 bytes             │
│ ExtensionData: variable                   │
└────────────────────────────────────────────┘

Extensions List:
┌────────────────────────────────────────────┐
│ Total Length: 2 bytes                      │
│ Extension 1: variable                      │
│ Extension 2: variable                      │
│ ...                                        │
│ Extension N: variable                      │
└────────────────────────────────────────────┘
```

### 扩展类型码

IANA 维护的 TLS 扩展类型码注册表：

| 扩展名称 | 类型码 | TLS 1.3 用途 |
|----------|--------|--------------|
| server_name | 0x0000 | 服务器名称指示（CH, EE） |
| max_fragment_length | 0x0001 | 最大片段长度（CH, EE） |
| status_request | 0x0005 | OCSP Stapling（CH, CR） |
| supported_groups | 0x000A | 支持的密钥交换组（CH, EE） |
| signature_algorithms | 0x000D | 支持的签名算法（CH, CR） |
| use_srtp | 0x000E | SRTP 协商（CH, EE） |
| heartbeat | 0x000F | 心跳（CH, EE） |
| application_layer_protocol_negotiation | 0x0010 | ALPN（CH, EE） |
| status_request_v2 | 0x0017 | OCSP v2（CH, CR） |
| extended_master_secret | 0x0017 | 扩展主密钥（CH, SH） |
| signed_certificate_timestamp | 0x0012 | SCT（CH, CR） |
| client_certificate_type | 0x0013 | 客户端证书类型（CH, EE） |
| server_certificate_type | 0x0014 | 服务器证书类型（CH, EE） |
| padding | 0x0015 | 填充（CH） |
| encrypt_then_mac | 0x0016 | EtM（CH, SH） |
| extended_master_secret | 0x0017 | EMS（CH, SH） |
| record_size_limit | 0x001C | 记录大小限制（CH, EE） |
| pre_shared_key | 0x0029 | PSK（CH, SH） |
| early_data | 0x002A | 早期数据（CH, EE） |
| supported_versions | 0x002B | 支持的版本（CH, SH） |
| cookie | 0x002C | Cookie（CH, HRR） |
| psk_key_exchange_modes | 0x002D | PSK 模式（CH） |
| certificate_authorities | 0x002F | CA（CH, CR） |
| oid_filters | 0x0030 | OID 过滤（CR） |
| post_handshake_auth | 0x0031 | 后握手认证（CH） |
| signature_algorithms_cert | 0x0032 | 证书签名算法（CH, CR） |
| key_share | 0x0033 | 密钥交换（CH, SH, HRR） |
| renegotiation_info | 0xFF01 | 重协商（CH, SH） |
| encrypted_client_hello | 0xFE0D | ECH（CH, EE） |

**消息指示**：
- CH: ClientHello
- SH: ServerHello
- EE: EncryptedExtensions
- CR: CertificateRequest
- HRR: HelloRetryRequest

### 关键扩展详解

#### server_name 扩展 (0x0000)

Server Name Indication (SNI) 用于指示要连接的服务器名称：

```
server_name 扩展格式：
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0000 (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ ServerNameList Length: 2 bytes            │
│ ServerName Type: 1 byte (0x00 = hostname)│
│ ServerName Length: 2 bytes                │
│ ServerName: variable (主机名)              │
└────────────────────────────────────────────┘

示例：
Host: example.com
┌────────────────────────────────────────────┐
│ 00 00 (ExtensionType)                      │
│ 00 0D (ExtensionData Length = 13)          │
│ 00 0B (ServerNameList Length = 11)         │
│ 00 (Name Type = hostname)                  │
│ 00 08 (Name Length = 8)                    │
│ 65 78 61 6D 70 6C 65 2E ("example.")       │
│ 63 6F 6D ("com")                           │
└────────────────────────────────────────────┘
```

**SNI 的作用**：
- 服务器根据 SNI 选择正确的证书
- 支持虚拟主机（多个域名共享 IP）
- 代理服务器可以根据 SNI 路由

#### supported_versions 扩展 (0x002B)

用于协商 TLS 版本（TLS 1.3 必需）：

```
ClientHello supported_versions:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x002B (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ SupportedVersions Length: 1 byte          │
│ Version 1: 2 bytes                         │
│ Version 2: 2 bytes                         │
│ ...                                        │
└────────────────────────────────────────────┘

ServerHello supported_versions:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x002B (2 bytes)           │
│ ExtensionData Length: 1 byte              │
│ SelectedVersion: 2 bytes                  │
└────────────────────────────────────────────┘

版本码：
- 0x0303: TLS 1.2
- 0x0304: TLS 1.3
```

**重要**：TLS 1.3 客户端必须在 ClientHello 中包含此扩展，ServerHello 必须包含选定的版本。

#### key_share 扩展 (0x0033)

携带密钥交换的公钥材料：

```
ClientHello key_share:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0033 (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ ClientShares Length: 2 bytes              │
│ KeyShareEntry 1:                           │
│   NamedGroup: 2 bytes                      │
│   KeyExchange Length: 2 bytes             │
│   KeyExchange: variable (公钥)             │
│ KeyShareEntry 2: ...                       │
└────────────────────────────────────────────┘

ServerHello key_share:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0033 (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ ServerShare:                               │
│   NamedGroup: 2 bytes                      │
│   KeyExchange Length: 2 bytes             │
│   KeyExchange: variable (服务器公钥)        │
└────────────────────────────────────────────┘

HelloRetryRequest key_share:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0033 (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ NamedGroup: 2 bytes (仅组名，无公钥)        │
└────────────────────────────────────────────┘
```

**公钥大小**：
- x25519: 32 字节
- x448: 56 字节
- secp256r1: 65 字节（未压缩点）
- secp384r1: 97 字节
- secp521r1: 133 字节

#### signature_algorithms 扩展 (0x000D)

声明支持的签名算法（TLS 1.3 必需）：

```
signature_algorithms 扩展格式：
┌────────────────────────────────────────────┐
│ ExtensionType: 0x000D (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ SupportedSignatureAlgorithms Length: 2 bytes│
│ SignatureScheme 1: 2 bytes                │
│ SignatureScheme 2: 2 bytes                │
│ ...                                        │
└────────────────────────────────────────────┘

常用签名算法：
┌────────────────────────────────────────────┐
│ 0x0403: ecdsa_secp256r1_sha256            │
│ 0x0503: ecdsa_secp384r1_sha384            │
│ 0x0603: ecdsa_secp521r1_sha512            │
│ 0x0804: rsa_pss_rsae_sha256              │
│ 0x0805: rsa_pss_rsae_sha384              │
│ 0x0806: rsa_pss_rsae_sha512              │
│ 0x0807: ed25519                           │
│ 0x0808: ed448                             │
└────────────────────────────────────────────┘
```

#### application_layer_protocol_negotiation 扩展 (0x0010)

应用层协议协商：

```
ALPN 扩展格式：
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0010 (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ ProtocolNameList Length: 2 bytes          │
│ ProtocolName 1 Length: 1 byte             │
│ ProtocolName 1: variable                   │
│ ProtocolName 2: ...                        │
└────────────────────────────────────────────┘

常用协议：
┌────────────────────────────────────────────┐
│ "h2" - HTTP/2                              │
│ "http/1.1" - HTTP/1.1                      │
│ "h3" - HTTP/3 (QUIC)                       │
└────────────────────────────────────────────┘
```

#### supported_groups 扩展 (0x000A)

声明支持的密钥交换组：

```
supported_groups 扩展格式：
┌────────────────────────────────────────────┐
│ ExtensionType: 0x000A (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ NamedGroupList Length: 2 bytes            │
│ NamedGroup 1: 2 bytes                      │
│ NamedGroup 2: 2 bytes                      │
│ ...                                        │
└────────────────────────────────────────────┘

常用组：
┌────────────────────────────────────────────┐
│ 0x0017: secp256r1 (P-256)                 │
│ 0x0018: secp384r1 (P-384)                 │
│ 0x0019: secp521r1 (P-521)                 │
│ 0x001D: x25519                            │
│ 0x001E: x448                              │
│ 0x0100: ffdhe2048                         │
│ 0x0101: ffdhe3072                         │
│ 0x0102: ffdhe4096                         │
└────────────────────────────────────────────┘
```

#### encrypted_client_hello 扪展 (0xFE0D)

加密 ClientHello（ECH）：

```
ECH 扩展格式：
┌────────────────────────────────────────────┐
│ ExtensionType: 0xFE0D (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ ECHConfig:                                 │
│   Public Key: variable                     │
│   Encoded ClientHello Inner: variable      │
└────────────────────────────────────────────┘

工作流程：
┌────────────────────────────────────────────┐
│ 1. 客户端获取 ECH 配置                      │
│ 2. 使用 ECH 公钥加密 ClientHello Inner     │
│ 3. 发送 ClientHello Outer (伪装)          │
│ 4. 服务器解密并处理 ClientHello Inner      │
│ 5. 返回 EncryptedExtensions (ECH 确认)     │
└────────────────────────────────────────────┘
```

**ECH 的作用**：
- 保护 SNI 不被被动观察者获取
- 保护其他敏感扩展
- 使 TLS 流量看起来像访问普通网站

#### pre_shared_key 扩展 (0x0029)

PSK 会话恢复：

```
ClientHello pre_shared_key:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0029 (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ PreSharedKeyList:                          │
│   PSKIdentityList Length: 2 bytes          │
│   PSKIdentity 1:                           │
│     Identity Length: 2 bytes              │
│     Identity: variable (票证)              │
│     ObliviousTicketAge: 4 bytes           │
│   ...                                      │
│ PSKBinderList Length: 2 bytes              │
│   BinderEntry 1: variable (HMAC)           │
│   ...                                      │
└────────────────────────────────────────────┘

ServerHello pre_shared_key:
┌────────────────────────────────────────────┐
│ ExtensionType: 0x0029 (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ SelectedIdentity: 2 bytes                 │
└────────────────────────────────────────────┘
```

#### cookie 扩展 (0x002C)

HelloRetryRequest cookie：

```
cookie 扩展格式：
┌────────────────────────────────────────────┐
│ ExtensionType: 0x002C (2 bytes)           │
│ ExtensionData Length: 2 bytes             │
│ Cookie: variable                           │
└────────────────────────────────────────────┘
```

客户端在重发 ClientHello 时必须包含此 cookie。

### EncryptedExtensions 消息

TLS 1.3 使用 EncryptedExtensions 消息传递服务器端的扩展响应：

```
EncryptedExtensions:
┌────────────────────────────────────────────┐
│ HandshakeType: 0x08 (1 byte)              │
│ Length: 3 bytes                           │
│ Extensions:                                │
│   - server_name (可选，确认)                │
│   - supported_groups (可选)               │
│   - alpn (可选，确认)                       │
│   - record_size_limit (可选)               │
│   - ...                                    │
└────────────────────────────────────────────┘
```

这些扩展在握手密钥建立后发送，内容被加密保护。

### 扩展处理规则

TLS 1.3 的扩展处理规则：

```
扩展处理规则：
┌────────────────────────────────────────────┐
│ 1. 必须处理所有已知扩展                      │
│ 2. 忽略不认识的扩展                          │
│ 3. 不能发送不认识的扩展                      │
│ 4. 扩展必须按规范出现在正确的消息中           │
│ 5. 扩展顺序不影响处理（TLS 1.3）             │
│ 6. 重复扩展可能导致错误                      │
└────────────────────────────────────────────┘
```

## 在 Prism 中的应用

### 扩展解析

Prism 解析 TLS 扩展用于提取关键信息：

```cpp
// 文件: src/prism/stealth/reality/request.cpp
auto parse_extensions(std::span<const std::byte> data)
    -> outcome::result<extensions_features>
{
    extensions_features features;
    size_t offset = 0;
    
    auto extensions_length = read_u16(data, offset);
    offset += 2;
    
    while (offset < extensions_length + 2) {
        auto extension_type = read_u16(data, offset);
        auto extension_length = read_u16(data, offset + 2);
        auto extension_data = data.subspan(offset + 4, extension_length);
        
        switch (extension_type) {
            case 0x0000: // server_name
                features.sni = parse_server_name(extension_data);
                break;
            case 0x002B: // supported_versions
                features.supported_versions = parse_supported_versions(extension_data);
                break;
            case 0x0033: // key_share
                features.key_shares = parse_key_share(extension_data);
                break;
            case 0x000D: // signature_algorithms
                features.signature_algorithms = parse_signature_algorithms(extension_data);
                break;
            case 0x0010: // alpn
                features.alpn = parse_alpn(extension_data);
                break;
            case 0x000A: // supported_groups
                features.supported_groups = parse_supported_groups(extension_data);
                break;
            case 0xFE0D: // ech
                features.ech = parse_ech(extension_data);
                break;
            default:
                // 忽略未知扩展
                break;
        }
        
        offset += 4 + extension_length;
    }
    
    return features;
}
```

### SNI 提取

Prism 提取 SNI 用于路由和匹配：

```cpp
// 文件: src/prism/stealth/reality/request.cpp
auto parse_server_name(std::span<const std::byte> data)
    -> outcome::result<std::string>
{
    size_t offset = 0;
    
    auto list_length = read_u16(data, offset);
    offset += 2;
    
    while (offset < list_length + 2) {
        auto name_type = static_cast<uint8_t>(data[offset]);
        offset += 1;
        
        auto name_length = read_u16(data, offset);
        offset += 2;
        
        if (name_type == 0x00) { // hostname
            std::string hostname;
            for (size_t i = 0; i < name_length; ++i) {
                hostname.push_back(static_cast<char>(data[offset + i]));
            }
            return hostname;
        }
        
        offset += name_length;
    }
    
    return ""; // 无 SNI
}
```

### key_share 提取

Prism 提取 key_share 用于密钥交换：

```cpp
// 文件: src/prism/stealth/reality/request.cpp
auto parse_key_share(std::span<const std::byte> data)
    -> outcome::result<std::vector<key_share_entry>>
{
    std::vector<key_share_entry> entries;
    size_t offset = 0;
    
    auto list_length = read_u16(data, offset);
    offset += 2;
    
    while (offset < list_length + 2) {
        auto named_group = read_u16(data, offset);
        auto key_length = read_u16(data, offset + 2);
        auto key_exchange = data.subspan(offset + 4, key_length);
        
        entries.push_back({
            .group = named_group,
            .key = std::vector<std::byte>(key_exchange.begin(), key_exchange.end())
        });
        
        offset += 4 + key_length;
    }
    
    return entries;
}
```

### ECH 解密

Prism 解密 ECH 扩展：

```cpp
// 文件: src/prism/stealth/ech/decrypt.cpp
auto decrypt_ech_payload(std::span<const std::byte> ech_data,
                         const ech_config& config)
    -> outcome::result<client_hello_features>
{
    // 使用 ECH 公钥对应的私钥解密
    auto decrypted = hpke_decrypt(config.private_key, ech_data);
    
    // 解解密的 ClientHello Inner
    return parse_client_hello(decrypted);
}
```

### Reality 扩展验证

Prism 的 Reality 模块验证扩展：

```cpp
// 文件: src/prism/stealth/reality/auth.cpp
auto match_server_name(const std::string& sni,
                       const reality_config& config)
    -> bool
{
    // 检查 SNI 是否在白名单中
    for (const auto& allowed : config.server_names) {
        if (sni == allowed) {
            return true;
        }
    }
    
    return false;
}
```

## 最佳实践

### 扩展顺序

1. **必需扩展优先**：supported_versions, key_share, signature_algorithms
2. **路由扩展其次**：server_name, alpn
3. **可选扩展最后**：padding, max_fragment_length

### 扩展解析

1. **边界检查**：检查长度和偏移量
2. **未知扩展**：忽略不认识的扩展
3. **重复处理**：处理重复扩展（使用最后一个）
4. **错误恢复**：优雅处理解析错误

### SNI 处理

1. **大小限制**：限制 SNI 长度（最大 255 字节）
2. **字符验证**：验证 SNI 字符的有效性
3. **编码处理**：处理 IDN 编码的域名
4. **隐私考虑**：考虑使用 ECH 保护 SNI

### key_share 处理

1. **组优先级**：优先使用 x25519
2. **公钥验证**：验证公钥的有效性
3. **HRR 处理**：正确处理 HelloRetryRequest
4. **密钥计算**：正确计算 ECDHE 共享密钥

## 常见问题

### Q1: TLS 1.3 有哪些必需扩展？

TLS 1.3 的必需扩展：

ClientHello 必须包含：
- supported_versions (0x002B)
- signature_algorithms (0x000D)
- key_share (0x0033) 或预声明 supported_groups

ServerHello 必须包含：
- supported_versions (0x002B)
- key_share (0x0033) 或 pre_shared_key (0x0029)

### Q2: 如何处理未知的扩展？

TLS 1.3 的扩展处理规则：

- **客户端**：可以发送未知扩展（但不应发送）
- **服务器**：必须忽略未知扩展
- **安全**：未知扩展不影响握手

### Q3: ECH 如何保护 SNI？

ECH 的工作原理：

1. 客户端获取服务器的 ECH 配置（公钥）
2. 使用 HPKE 加密 ClientHello Inner（包含真实 SNI）
3. 发送 ClientHello Outer（包含伪装 SNI）
4. 服务器解密并处理真实 ClientHello

这样被动观察者只能看到伪装的 SNI。

### Q4: ALPN 协商失败会发生什么？

如果 ALPN 协商失败：

- 客户端和服务器没有共同支持的协议
- 服务器不回复 ALPN 扩展
- 客户端可以选择继续（使用默认协议）或中止连接

### Q5: 为什么有些扩展在 EncryptedExtensions 中？

EncryptedExtensions 消息的作用：

- 提供隐私保护
- 防止被动攻击者获取敏感信息
- 包含路由相关信息（server_name, alpn）

### Q6: supported_groups 和 key_share 的关系？

两者配合使用：

- supported_groups：声明支持的密钥交换组
- key_share：提供实际的公钥材料

客户端可以在 supported_groups 中声明更多组，但在 key_share 中只提供最可能被接受的公钥。

### Q7: Prism 如何检测 TLS 指纹？

Prism 通过扩展特征检测：

- 扩展顺序和组合
- cipher_suites 顺序
- supported_groups 顺序
- 与已知指纹库匹配

### Q8: 如何添加新的 TLS 扩展？

添加新扩展的步骤：

1. 定义扩展类型码（可使用临时码或申请正式码）
2. 设计扩展数据格式
3. 实现解析和处理逻辑
4. 测试兼容性
5. 向 IANA 申请正式注册

## 参考资料

- [RFC 8446 Section 4.2 - Extensions](https://tools.ietf.org/html/rfc8446#section-4.2)
- [IANA TLS Extension Type Values Registry](https://www.iana.org/assignments/tls-extensiontype-values/)
- [RFC 6066 - TLS Extensions](https://tools.ietf.org/html/rfc6066)
- [RFC 8499 - ECH Specification](https://tools.ietf.org/html/rfc8499)

## 相关知识

- [[ref/protocol/tls-clienthello|TLS ClientHello]] — ClientHello 消息详解
- [[ref/protocol/tls-serverhello|TLS ServerHello]] — ServerHello 消息详解
- [[ref/protocol/tls-1-3|TLS 1.3]] — TLS 1.3 协议详解
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — JA3/JA4 指纹
- [[ref/stealth/reality|Reality 协议]] — Reality 身份检测
- [[ref/stealth/ech|ECH]] — 加密 ClientHello