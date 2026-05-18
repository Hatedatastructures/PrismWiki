---
title: "TLS ServerRandom 字段"
category: "protocol"
type: ref
module: ref
source: "RFC 8446 Section 4.1.3"
tags: [协议, tls, serverrandom, 密钥派生, 密钥, 降级保护, shadowtls]
created: 2026-05-15
updated: 2026-05-17
related: [tls-serverhello, tls-1-3, hkdf, shadowtls, sha256]
layer: ref
---

# TLS ServerRandom 字段

**类别**: 协议

## 概述

TLS ServerRandom 是 ServerHello 消息中的 32 字节随机数字段，由服务器生成并发送给客户端。ServerRandom 在 TLS 协议中扮演着关键的密钥派生角色，与 ClientHello 中的 ClientRandom 一起参与会话密钥的计算。在 TLS 1.3 中，ServerRandom 还承担了降级保护的功能，通过在特定位置嵌入降级标记来防止版本降级攻击。

ServerRandom 字段在代理伪装领域有着重要的应用：

- **密钥派生**：参与 TLS 1.3 的 HKDF 密钥派生过程
- **ShadowTLS 协议**：使用 ServerRandom 生成写入密钥和 HMAC 密钥
- **降级检测**：通过检查 ServerRandom 的降级标记检测攻击

### 历史演变

ServerRandom 字段的演变：

- **SSL 3.0**：ServerRandom 为 32 字节，前 4 字节为 GMT Unix 时间
- **TLS 1.0-1.2**：延续 SSL 3.0 的格式，前 4 字节为时间戳
- **TLS 1.3**：完全随机，移除时间戳，增加降级保护标记

### ServerRandom 与 ClientRandom

ServerRandom 和 ClientRandom 共同参与密钥派生：

```
密钥派生中的角色：

TLS 1.2 (PRF):
┌────────────────────────────────────────────┐
│ Master Secret = PRF(pre_master_secret,    │
│                   "master secret",         │
│                   ClientRandom +          │
│                   ServerRandom)           │
└────────────────────────────────────────────┘

TLS 1.3 (HKDF):
┌────────────────────────────────────────────┐
│ Transcript Hash 包含：                      │
│   ClientHello (含 ClientRandom)            │
│   ServerHello (含 ServerRandom)            │
│                                            │
│ Handshake Secret = HKDF-Extract(          │
│   derived_secret,                          │
│   ECDHE_shared_secret)                     │
│                                            │
│ Traffic Secret = HKDF-Expand-Label(        │
│   handshake_secret,                        │
│   label,                                   │
│   transcript_hash)                         │
└────────────────────────────────────────────┘
```

### 字段位置

ServerRandom 在 ServerHello 消息中的位置：

```
ServerHello:
┌────────────────────────────────────────────┐
│ legacy_version: 2 bytes                    │
│ random: 32 bytes ← ServerRandom           │
│ legacy_session_id_echo: 0-32 bytes        │
│ cipher_suite: 2 bytes                     │
│ legacy_compression_method: 1 byte         │
│ extensions: variable                       │
└────────────────────────────────────────────┘
```

## 协议详解

### 字段结构

ServerRandom 字段的二进制结构：

```
ServerRandom (TLS 1.2):
┌────────────────────────────────────────────┐
│ gmt_unix_time: 4 bytes                     │
│ random_bytes: 28 bytes                     │
└────────────────────────────────────────────┘

ServerRandom (TLS 1.3):
┌────────────────────────────────────────────┐
│ random_bytes: 32 bytes                     │
│ (完全随机，无时间戳)                         │
│                                            │
│ 特殊情况（降级保护）：                        │
│ random_bytes[24:32] = 降级标记              │
└────────────────────────────────────────────┘
```

#### TLS 1.2 格式（遗留）

在 TLS 1.2 中，ServerRandom 的前 4 字节为 GMT Unix 时间戳：

```
gmt_unix_time 字段：
┌────────────────────────────────────────────┐
│ 当前时间的 Unix 时间戳（32-bit）             │
│ 注意：某些实现可能忽略此字段                 │
└────────────────────────────────────────────┘

目的：
1. 增加随机性来源
2. 协助调试（记录握手时间）
3. 防止某些弱随机数生成器的问题
```

**注意**：TLS 1.3 明确移除了时间戳字段，整个 ServerRandom 为完全随机数据。

#### TLS 1.3 格式

TLS 1.3 中 ServerRandom 为 32 字节的完全随机数据：

```
生成要求：
┌────────────────────────────────────────────┐
│ 必须使用加密安全的随机数生成器               │
│ 推荐使用：                                  │
│   - OpenSSL RAND_bytes                     │
│   -系统提供的 CSPRNG                        │
│   - /dev/urandom                           │
│                                            │
│ 禁止使用：                                  │
│   - 时间戳                                  │
│   - 伪随机数生成器（如 rand()）              │
│   - 可预测的序列                            │
└────────────────────────────────────────────┘
```

### 降级保护机制

TLS 1.3 在 ServerRandom 中嵌入降级保护标记：

#### 降级场景

当服务器协商的版本低于客户端支持的版本时，会在 ServerRandom 的最后 8 字节设置降级标记：

```
降级场景：
┌────────────────────────────────────────────┐
│ 客户端支持 TLS 1.3                         │
│ 服务器支持 TLS 1.2                         │
│ 协商结果：TLS 1.2                          │
│                                            │
│ 服务器设置降级标记                          │
│ ServerRandom[24:32] = "DOWNGRD\x01"       │
└────────────────────────────────────────────┘
```

#### 降级标记值

降级标记的具体值：

```
降级标记（ServerRandom[24:32]）：

TLS 1.2 降级：
│ 44 4F 57 4E 47 52 44 01                  │
│ D  O  W  N  G  R  D  \x01               │
│ "DOWNGRD" + 0x01                         │

TLS 1.1 或更低降级：
│ 44 4F 57 4E 47 52 44 00                  │
│ D  O  W  N  G  R  D  \x00               │
│ "DOWNGRD" + 0x00                         │

正常 TLS 1.3（无降级）：
│ 随机数据（不等于上述标记）                  │
```

#### 客户端检测逻辑

客户端检测降级标记的逻辑：

```cpp
// 降级检测示例
auto check_downgrade(std::span<const std::byte> server_random) -> downgrade_status
{
    if (server_random.size() != 32) {
        return downgrade_status::invalid;
    }
    
    // 检查 TLS 1.2 降级标记
    constexpr std::array<std::byte, 8> downgrade_tls12 = {
        std::byte{0x44}, std::byte{0x4F}, std::byte{0x57}, std::byte{0x4E},
        std::byte{0x47}, std::byte{0x52}, std::byte{0x44}, std::byte{0x01}
    };
    
    // 检查 TLS 1.1 或更低降级标记
    constexpr std::array<std::byte, 8> downgrade_tls11 = {
        std::byte{0x44}, std::byte{0x4F}, std::byte{0x57}, std::byte{0x4E},
        std::byte{0x47}, std::byte{0x52}, std::byte{0x44}, std::byte{0x00}
    };
    
    auto tail = server_random.subspan<24, 8>();
    
    if (std::equal(tail.begin(), tail.end(), downgrade_tls12.begin())) {
        return downgrade_status::tls12_downgrade;
    }
    
    if (std::equal(tail.begin(), tail.end(), downgrade_tls11.begin())) {
        return downgrade_status::tls11_downgrade;
    }
    
    return downgrade_status::no_downgrade;
}
```

#### HelloRetryRequest Random

HelloRetryRequest 使用固定的 ServerRandom 值：

```
HelloRetryRequest ServerRandom（固定值）：
CF 21 AD 74 E5 9A 61 11
BE 1D 8C 02 1E 65 B8 91
C2 A2 11 16 7A BB 8C 5E
07 9E 09 E2 C8 A8 33 9C

计算方式：
SHA-256("HelloRetryRequest")
```

这个固定值用于区分 HelloRetryRequest 和正常的 ServerHello。

### ShadowTLS 的 ServerRandom 利用

ShadowTLS 协议使用 ServerRandom 生成密钥：

#### 写入密钥生成

ShadowTLS 使用 ServerRandom 和密码生成写入密钥：

```
写入密钥计算：
┌────────────────────────────────────────────┐
│ WriteKey = SHA256(password + serverRandom)│
│                                            │
│ 输入：                                      │
│   password: ShadowTLS 配置的密码            │
│   serverRandom: ServerHello 的 32 字节     │
│                                            │
│ 输出：                                      │
│   WriteKey: 32 字节的 SHA-256 输出          │
└────────────────────────────────────────────┘
```

#### HMAC 密钥生成

ShadowTLS 使用 ServerRandom 计算 HMAC：

```
数据帧 HMAC 计算：

验证帧（接收）：
┌────────────────────────────────────────────┐
│ HMAC_Verify = HMAC-SHA1(password,         │
│                         serverRandom +    │
│                         "C" +             │
│                         payload)[:4]      │
│                                            │
│ "C" 表示客户端数据                          │
└────────────────────────────────────────────┘

发送帧（写入）：
┌────────────────────────────────────────────┐
│ HMAC_Write = HMAC-SHA1(password,          │
│                        serverRandom +     │
│                        payload)[:4]       │
│                                            │
│ 注意：写入帧不包含 "C" 前缀                  │
└────────────────────────────────────────────┘
```

#### ShadowTLS v3 流程

ShadowTLS v3 的 ServerRandom 使用流程：

```
ShadowTLS v3 流程：

Client                                    Server
  |                                          |
  |---------- ClientHello ------------------>|
  |   session_id: [HMAC 标签]                 |
  |                                          |
  |<--------- ServerHello -------------------|
  |   random: [serverRandom]                 |
  |                                          |
  |   ShadowTLS 计算：                        |
  |   WriteKey = SHA256(pass + serverRandom)│
  |                                          |
  |<========= ShadowTLS Data Frames =========|
  |   每帧包含 HMAC 标签                       │
  |                                          |
```

### TLS 1.3 密钥派生流程

ServerRandom 在 TLS 1.3 密钥派生中的作用：

```
TLS 1.3 密钥派生完整流程：

1. Transcript 计算：
   transcript_hash_1 = Hash(ClientHello)
   transcript_hash_2 = Hash(ClientHello + ServerHello)
   
   ServerHello 包含 ServerRandom

2. Early Secret：
   early_secret = HKDF-Extract(0, 0^hash_len)
   derived_secret = HKDF-Expand-Label(early_secret, "derived", "")

3. Handshake Secret：
   handshake_secret = HKDF-Extract(derived_secret, ECDHE_shared_secret)
   
   client_hs_secret = HKDF-Expand-Label(handshake_secret,
                                         "c hs traffic",
                                         transcript_hash_2)
   
   server_hs_secret = HKDF-Expand-Label(handshake_secret,
                                         "s hs traffic",
                                         transcript_hash_2)

4. Master Secret：
   master_secret = HKDF-Extract(derived_secret_2, 0^hash_len)
   
   client_app_secret = HKDF-Expand-Label(master_secret,
                                          "c ap traffic",
                                          transcript_hash_final)
   
   server_app_secret = HKDF-Expand-Label(master_secret,
                                          "s ap traffic",
                                          transcript_hash_final)
```

**关键点**：ServerRandom 通过 transcript_hash 参与密钥派生。

## 在 Prism 中的应用

### ServerRandom 提取

Prism 从 ServerHello 中提取 ServerRandom：

```cpp
// 文件: src/prism/stealth/reality/request.cpp
auto parse_server_hello(std::span<const std::byte> data)
    -> outcome::result<server_hello_features>
{
    size_t offset = 0;
    
    // legacy_version: 2 bytes
    offset += 2;
    
    // random: 32 bytes ← ServerRandom
    auto server_random = data.subspan(offset, 32);
    offset += 32;
    
    // 检查降级标记
    auto downgrade_status = check_downgrade(server_random);
    
    // session_id_echo
    auto session_id_length = static_cast<size_t>(data[offset]);
    offset += 1 + session_id_length;
    
    // cipher_suite: 2 bytes
    auto cipher_suite = read_u16(data, offset);
    
    return server_hello_features{
        .random = server_random,
        .downgrade_status = downgrade_status,
        .cipher_suite = cipher_suite,
        // ...
    };
}
```

### ShadowTLS 密钥计算

Prism 的 ShadowTLS 模块使用 ServerRandom 计算密钥：

```cpp
// 文件: src/prism/stealth/shadowtls/auth.cpp
auto compute_write_key(std::string_view password,
                       std::span<const std::byte> server_random)
    -> std::vector<std::byte>
{
    // WriteKey = SHA256(password + serverRandom)
    std::vector<std::byte> input;
    
    // 添加密码
    input.reserve(password.size() + server_random.size());
    for (char c : password) {
        input.push_back(static_cast<std::byte>(c));
    }
    
    // 添加 ServerRandom
    input.append(server_random.begin(), server_random.end());
    
    // 计算 SHA-256
    return sha256(input);
}

auto verify_frame_hmac(std::string_view password,
                       std::span<const std::byte> server_random,
                       std::span<const std::byte> payload,
                       std::span<const std::byte> received_hmac)
    -> bool
{
    // HMAC_Verify = HMAC-SHA1(password, serverRandom + "C" + payload)[:4]
    std::vector<std::byte> hmac_input;
    
    // serverRandom
    hmac_input.append(server_random.begin(), server_random.end());
    
    // "C" 标记（客户端数据）
    hmac_input.push_back(static_cast<std::byte>('C'));
    
    // payload
    hmac_input.append(payload.begin(), payload.end());
    
    // 计算 HMAC-SHA1
    auto hmac = hmac_sha1(password, hmac_input);
    
    // 取前 4 字节比较（常量时间）
    return constant_time_compare(received_hmac, hmac.subspan(0, 4));
}
```

### Reality 密钥派生

Prism 的 Reality 模块使用 ServerRandom（通过 transcript）：

```cpp
// 文件: src/prism/stealth/reality/keygen.cpp
auto derive_handshake_keys(const std::vector<std::byte>& shared_secret,
                           const std::vector<std::byte>& client_hello,
                           const std::vector<std::byte>& server_hello)
    -> handshake_keys
{
    // transcript_hash 包含 ServerHello（含 ServerRandom）
    auto transcript = concatenate(client_hello, server_hello);
    auto transcript_hash = sha256(transcript);
    
    // 派生握手密钥
    auto early_secret = hkdf_extract(/* zero */, /* zero */);
    auto derived_secret = hkdf_expand_label(early_secret, "derived", {});
    auto handshake_secret = hkdf_extract(derived_secret, shared_secret);
    
    // 计算双向密钥
    auto client_secret = hkdf_expand_label(handshake_secret,
                                            "c hs traffic",
                                            transcript_hash);
    auto server_secret = hkdf_expand_label(handshake_secret,
                                            "s hs traffic",
                                            transcript_hash);
    
    return handshake_keys{
        .client_key = derive_key(client_secret),
        .client_iv = derive_iv(client_secret),
        .server_key = derive_key(server_secret),
        .server_iv = derive_iv(server_secret)
    };
}
```

### ServerRandom 生成

Prism 生成安全的 ServerRandom：

```cpp
// 文件: src/prism/stealth/reality/response.cpp
auto generate_server_random(bool is_hrr = false)
    -> std::vector<std::byte>
{
    std::vector<std::byte> random(32);
    
    if (is_hrr) {
        // HelloRetryRequest 使用固定值
        constexpr std::array<std::byte, 32> hrr_random = {
            // SHA-256("HelloRetryRequest")
            std::byte{0xCF}, std::byte{0x21}, std::byte{0xAD}, std::byte{0x74},
            std::byte{0xE5}, std::byte{0x9A}, std::byte{0x61}, std::byte{0x11},
            std::byte{0xBE}, std::byte{0x1D}, std::byte{0x8C}, std::byte{0x02},
            std::byte{0x1E}, std::byte{0x65}, std::byte{0xB8}, std::byte{0x91},
            std::byte{0xC2}, std::byte{0xA2}, std::byte{0x11}, std::byte{0x16},
            std::byte{0x7A}, std::byte{0xBB}, std::byte{0x8C}, std::byte{0x5E},
            std::byte{0x07}, std::byte{0x9E}, std::byte{0x09}, std::byte{0xE2},
            std::byte{0xC8}, std::byte{0xA8}, std::byte{0x33}, std::byte{0x9C}
        };
        std::copy(hrr_random.begin(), hrr_random.end(), random.begin());
    } else {
        // 正常 ServerHello 使用随机值
        RAND_bytes(reinterpret_cast<unsigned char*>(random.data()), 32);
    }
    
    return random;
}
```

## 最佳实践

### ServerRandom 生成

1. **加密安全随机**：使用 CSPRNG（如 OpenSSL RAND_bytes）
2. **完全随机**：TLS 1.3 不使用时间戳
3. **避免模式**：不应有可预测的模式
4. **降级标记**：仅在必要时设置降级标记

### 降级检测

1. **客户端验证**：检查 ServerRandom 的降级标记
2. **安全策略**：对降级攻击采取适当的响应
3. **日志记录**：记录降级事件用于分析
4. **用户通知**：可选地通知用户潜在的降级风险

### ShadowTLS 密钥使用

1. **密钥安全**：WriteKey 应安全存储
2. **HMAC 验证**：使用常量时间比较
3. **帧处理**：正确处理带 HMAC 的数据帧
4. **密钥更新**：定期更新密钥

### HelloRetryRequest 处理

1. **固定值识别**：使用固定的 ServerRandom 识别 HRR
2. **重新计算**：重新发送 ClientHello 时更新 transcript
3. **Cookie 处理**：正确处理 HRR 的 cookie 扩展
4. **限制次数**：限制 HRR 的次数

## 常见问题

### Q1: ServerRandom 为什么从 32 字节变为完全随机？

TLS 1.3 移除时间戳的原因：

1. **时间戳问题**：时间戳可能泄露服务器信息
2. **随机性**：时间戳不够随机，可能被预测
3. **简化实现**：简化随机数生成逻辑
4. **安全增强**：完全随机更安全

### Q2: 降级保护标记是如何工作的？

降级保护机制：

1. 服务器协商了低于客户端支持的版本
2. 服务器在 ServerRandom 最后 8 字节设置标记
3. 客户端检测标记，发现降级
4. 客户端可以警告用户或拒绝连接

这防止了主动攻击者强制降级 TLS 版本。

### Q3: HelloRetryRequest 的 ServerRandom 是什么？

HelloRetryRequest 使用固定值：

```
固定值 = SHA-256("HelloRetryRequest")
```

这个值用于区分 HelloRetryRequest 和正常 ServerHello，避免客户端混淆。

### Q4: ShadowTLS 为什么使用 SHA256(password + serverRandom)？

ShadowTLS 的密钥生成方式：

1. **简洁**：简单且高效的密钥生成
2. **安全**：使用 SHA-256 的哈希安全性
3. **可验证**：客户端可以独立计算相同密钥
4. **绑定**：密钥绑定到特定会话（serverRandom）

### Q5: 如果 ServerRandom 可预测会发生什么？

可预测的 ServerRandom 会带来严重安全风险：

1. **密钥泄露**：攻击者可能预测密钥
2. **会话破解**：可能解密通信内容
3. **降级绕过**：可能绕过降级检测
4. **HMAC 伪造**：ShadowTLS 的 HMAC 可能被伪造

### Q6: Prism 如何处理 HRR ServerRandom？

Prism 处理 HelloRetryRequest：

1. 检测固定的 ServerRandom 值
2. 解析 cookie 扩展
3. 重新生成 ClientHello（带正确的 key_share 和 cookie）
4. 重新计算 transcript hash
5. 继续握手流程

### Q7: ServerRandom 与 ClientRandom 的区别？

主要区别：

| 特性 | ClientRandom | ServerRandom |
|------|--------------|--------------|
| 生成方 | 客户端 | 服务器 |
| 位置 | ClientHello | ServerHello |
| 降级标记 | 无 | 有 |
| HRR 固定值 | 无 | 有 |
| 密钥派生 | 参与 | 参与 |

### Q8: 如何验证 ServerRandom 的有效性？

验证 ServerRandom：

1. **长度检查**：必须为 32 字节
2. **降级检查**：检查降级标记
3. **HRR 检查**：检查是否为 HRR 固定值
4. **随机性**：不应有明显的模式（非降级情况）

## 参考资料

- [RFC 8446 Section 4.1.3 - ServerHello](https://tools.ietf.org/html/rfc8446#section-4.1.3)
- [RFC 8446 Section 4.1.4 - HelloRetryRequest](https://tools.ietf.org/html/rfc8446#section-4.1.4)
- [RFC 8446 Section 7.4 - Key Derivation](https://tools.ietf.org/html/rfc8446#section-7.4)
- [ShadowTLS Protocol Specification](https://github.com/ihciah/shadowtls)

## 相关知识

- [[ref/protocol/tls-serverhello|TLS ServerHello]] — ServerHello 消息详解
- [[ref/protocol/tls-1-3|TLS 1.3]] — TLS 1.3 协议详解
- [[ref/crypto/hkdf|HKDF]] — TLS 1.3 密钥派生
- [[ref/crypto/sha256|SHA-256]] — ShadowTLS 使用的哈希
- [[ref/crypto/hmac-sha1|HMAC-SHA1]] — ShadowTLS HMAC 计算
- [[ref/stealth/shadowtls|ShadowTLS 协议]] — ShadowTLS 伪装协议