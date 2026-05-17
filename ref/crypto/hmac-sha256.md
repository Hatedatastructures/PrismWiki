---
title: "HMAC-SHA256"
category: "crypto"
type: ref
module: ref
source: "RFC 2104, RFC 6234"
tags: [密码学, hmac, sha256, 消息认证码]
created: 2026-05-15
updated: 2026-05-17
---

# HMAC-SHA256

**类别**: 密码学

## 概述

HMAC-SHA256（Hash-based Message Authentication Code with SHA-256）是一种基于 SHA-256 哈希函数的消息认证码算法。HMAC 由 Bellare、Canetti 和 Krawczyk 于 1996 年提出并由 RFC 2104 标准化，HMAC-SHA256 特指使用 SHA-256 作为底层哈希函数的实现。它生成 256 位（32 字节）的消息认证码，是目前最广泛使用的 MAC 算法之一。

HMAC-SHA256 结合了 HMAC 构造的安全性和 SHA-256 哈希函数的强度。HMAC 的设计提供了一个通用的 MAC 框架，可以使用任意安全的哈希函数作为底层原语。SHA-256 作为底层哈希函数，提供了 256 位的输出长度和经过验证的安全保证。这种组合使得 HMAC-SHA256 成为现代密码学应用的首选 MAC 算法。

HMAC-SHA256 在 TLS 1.2 和 TLS 1.3 中都有重要地位。在 TLS 1.2 中，HMAC-SHA256 用于记录层的数据认证（作为密码套件的 HMAC）。在 TLS 1.3 中，HMAC-SHA256 是 HKDF 的底层原语，用于所有密钥派生操作，包括握手密钥和应用数据密钥的派生。

HMAC-SHA256 的核心功能是验证消息的完整性和真实性。完整性确保消息没有被篡改，真实性确保消息来自声称的发送者。这通过密钥与消息的混合来实现：只有持有正确密钥的实体才能生成有效的 HMAC 值，攻击者无法在没有密钥的情况下伪造有效的认证码。

HMAC-SHA256 的工作原理是通过两次哈希运算来构造认证码。首先，将密钥与内部填充值（ipad）异或，然后与消息一起进行第一次哈希运算。接着，将密钥与外部填充值（opad）异或，并与第一次哈希的结果一起进行第二次哈希运算。这种嵌套结构提供了额外的安全保障，防止长度扩展攻击和其他针对底层哈希函数的攻击。

HMAC-SHA256 的主要优势包括：
- 高安全性：256 位输出提供 256 位 MAC 安全强度
- 广泛支持：几乎所有加密库和协议都支持
- 简单实现：只需调用底层哈希函数两次
- 恒定时间：现代实现支持恒定时间操作
- 抗长度扩展：HMAC 构造天然防御此攻击

在 HKDF 中，HMAC-SHA256 是提取和扩展两个阶段的核心原语。提取阶段使用 HMAC-SHA256 从输入密钥材料中生成伪随机密钥（PRK），扩展阶段使用 HMAC-SHA256 的迭代调用从 PRK 中派生输出密钥材料。这种密钥派生方法在 TLS 1.3 中广泛使用。

HMAC-SHA256 在 TLS 1.3 的密码套件中也有直接使用。TLS_AES_128_GCM_SHA256 和 TLS_CHACHA20_POLY1305_SHA256 都使用 SHA-256 作为握手哈希和 HKDF 原语，这意味着 HMAC-SHA256 在 TLS 1.3 中无处不在。

在代理服务器和网络协议实现中，HMAC-SHA256 用于多种场景：密钥派生、消息认证、Finished 消息验证等。Prism 在 Reality 协议的密钥派生、ShadowTLS 的密钥计算等多个场景中使用 HMAC-SHA256。它提供了安全、高效的认证和密钥派生基础。

总的来说，HMAC-SHA256 是现代密码学中最重要的 MAC 算法之一。它的安全性经过严格证明，实现简单高效，在 TLS、SSH、IPsec 等协议中广泛使用。理解 HMAC-SHA256 的工作原理和正确实现方法对于构建安全的通信系统至关重要。

## 算法原理

### HMAC 结构

HMAC（Hash-based Message Authentication Code）是一种基于加密哈希函数的消息认证码构造。HMAC 的核心思想是利用哈希函数的压缩特性，将密钥和消息混合在一起，生成固定长度的认证标签。

HMAC 的通用公式为：

```
HMAC(K, M) = H((K ⊕ opad) || H((K ⊕ ipad) || M))
```

其中：
- H 是加密哈希函数（在 HMAC-SHA256 中为 SHA-256）
- K 是密钥
- M 是消息
- ipad 是内部填充值（0x36 重复块大小次）
- opad 是外部填充值（0x5C 重复块大小次）
- || 表示连接操作
- ⊕ 表示异或操作

对于 HMAC-SHA256，具体的参数如下：
- 块大小（B）= 64 字节（512 位）
- 输出大小（L）= 32 字节（256 位）
- ipad = 0x36 重复 64 次 = 36363636...36（64 字节）
- opad = 0x5C 重复 64 次 = 5C5C5C5C...5C（64 字节）

### HMAC-SHA256 完整算法

**输入**：密钥 K（任意长度），消息 M（任意长度）

**输出**：256 位（32 字节）的消息认证码

**步骤**：

1. **密钥预处理**：
   - 如果 |K| > 64 字节：K' = SHA256(K)
   - 如果 |K| < 64 字节：K' = K || 00...00（用零填充到 64 字节）
   - 如果 |K| = 64 字节：K' = K

2. **计算内部哈希**：
   - 计算内部填充密钥：Ki = K' ⊕ ipad = K' ⊕ (0x36 || 0x36 || ... || 0x36)
   - 计算内部哈希：Hi = SHA256(Ki || M)

3. **计算外部哈希**：
   - 计算外部填充密钥：Ko = K' ⊕ opad = K' ⊕ (0x5C || 0x5C || ... || 0x5C)
   - 计算最终 HMAC：HMAC-SHA256(K, M) = SHA256(Ko || Hi)

### SHA-256 哈希函数

SHA-256 是 SHA-2 家族的成员，生成 256 位（32 字节）的哈希值。

**处理流程**：

1. **消息填充**：
   - 在消息末尾添加一个 1 位
   - 添加若干个 0 位，使消息长度为 448 mod 512
   - 在末尾附加 64 位（8 字节）的原始消息长度（大端序）

2. **初始化哈希值**：
   ```
   H0 = 0x6a09e667
   H1 = 0xbb67ae85
   H2 = 0x3c6ef372
   H3 = 0xa54ff53a
   H4 = 0x510e527f
   H5 = 0x9b05688c
   H6 = 0x1f83d9ab
   H7 = 0x5be0cd19
   ```

3. **消息扩展**：
   - 将每个 512 位块扩展为 64 个 32 位字

4. **64 轮压缩**：
   - 使用轮函数更新工作变量
   - 使用 64 个轮常量

5. **输出**：
   - 256 位哈希值

**安全性**：SHA-256 目前被认为是安全的，没有已知的实际攻击。

### 安全性分析

**HMAC 安全性证明**

Bellare、Canetti 和 Krawczyk 在 1996 年证明了 HMAC 的安全性可以归约到压缩函数的伪随机性。这意味着如果压缩函数是安全的 PRF，那么 HMAC 也是安全的。

对于 HMAC-SHA256：
- 安全性基于 SHA-256 压缩函数作为伪随机函数的性质
- 不依赖于 SHA-256 的抗碰撞性（这与 HMAC-SHA1 不同）

**安全强度**

| 属性 | HMAC-SHA256 | HMAC-SHA1 |
|------|-------------|-----------|
| 输出长度 | 256 位 | 160 位 |
| 伪造攻击复杂度 | 2^256 | 2^160 |
| 密钥恢复复杂度 | 2^L | 2^L |
| 安全强度 | 256 位 | 160 位 |

**抗攻击性**

HMAC-SHA256 的抗攻击性：

1. **抗长度扩展攻击**：
   - HMAC 的嵌套结构天然防御
   - 攻击者无法从 HMAC 值推断内部哈希

2. **抗碰撞攻击**：
   - HMAC 不依赖于哈希函数的抗碰撞性
   - 即使 SHA-256 有碰撞攻击，HMAC-SHA256 仍安全

3. **抗伪造攻击**：
   - 需要约 2^256 次操作伪造 HMAC
   - 远超实际计算能力

4. **抗密钥恢复攻击**：
   - 需要穷举密钥空间
   - 对于 256 位密钥，需要约 2^256 次操作

### 完整实现示例

```cpp
// HMAC-SHA256 完整实现
std::array<std::uint8_t, 32> hmac_sha256(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> message) {
    
    constexpr size_t block_size = 64;
    constexpr std::uint8_t ipad_value = 0x36;
    constexpr std::uint8_t opad_value = 0x5C;
    
    // 步骤 1：密钥预处理
    std::vector<std::uint8_t> processed_key(block_size, 0);
    
    if (key.size() > block_size) {
        // 密钥过长，先哈希
        auto key_hash = sha256(key);
        std::copy(key_hash.begin(), key_hash.end(), processed_key.begin());
    } else {
        // 密钥较短，用零填充
        std::copy(key.begin(), key.end(), processed_key.begin());
    }
    
    // 步骤 2：计算内部哈希
    std::vector<std::uint8_t> inner_input(block_size);
    for (size_t i = 0; i < block_size; ++i) {
        inner_input[i] = processed_key[i] ^ ipad_value;
    }
    inner_input.insert(inner_input.end(), message.begin(), message.end());
    auto inner_hash = sha256(inner_input);
    
    // 步骤 3：计算外部哈希
    std::vector<std::uint8_t> outer_input(block_size);
    for (size_t i = 0; i < block_size; ++i) {
        outer_input[i] = processed_key[i] ^ opad_value;
    }
    outer_input.insert(outer_input.end(), inner_hash.begin(), inner_hash.end());
    
    return sha256(outer_input);
}
```

### OpenSSL/BoringSSL 实现

```cpp
// 使用 OpenSSL/BoringSSL
std::array<std::uint8_t, 32> hmac_sha256(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> message) {
    
    std::array<std::uint8_t, 32> result;
    unsigned int result_len = 32;
    
    HMAC(EVP_sha256(), key.data(), key.size(),
         message.data(), message.size(),
         result.data(), &result_len);
    
    return result;
}

// 使用 EVP 接口（更灵活）
std::array<std::uint8_t, 32> hmac_sha256_evp(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> message) {
    
    HMAC_CTX* ctx = HMAC_CTX_new();
    HMAC_Init_ex(ctx, key.data(), key.size(), EVP_sha256(), nullptr);
    HMAC_Update(ctx, message.data(), message.size());
    
    std::array<std::uint8_t, 32> result;
    HMAC_Final(ctx, result.data(), nullptr);
    
    HMAC_CTX_free(ctx);
    
    return result;
}
```

### 增量 HMAC

```cpp
// 增量 HMAC-SHA256
class incremental_hmac_sha256 {
public:
    incremental_hmac_sha256(std::span<const std::uint8_t> key) {
        // 预处理密钥
        std::vector<std::uint8_t> processed_key(64, 0);
        if (key.size() > 64) {
            auto key_hash = sha256(key);
            std::copy(key_hash.begin(), key_hash.end(), processed_key.begin());
        } else {
            std::copy(key.begin(), key.end(), processed_key.begin());
        }
        
        // 初始化内部哈希
        inner_hasher_.init();
        
        std::array<std::uint8_t, 64> inner_key;
        for (size_t i = 0; i < 64; ++i) {
            inner_key[i] = processed_key[i] ^ 0x36;
        }
        inner_hasher_.update(inner_key);
        
        // 保存外部密钥
        for (size_t i = 0; i < 64; ++i) {
            outer_key_[i] = processed_key[i] ^ 0x5C;
        }
    }
    
    void update(std::span<const std::uint8_t> data) {
        inner_hasher_.update(data);
    }
    
    std::array<std::uint8_t, 32> finalize() {
        // 完成内部哈希
        auto inner_result = inner_hasher_.finalize();
        
        // 计算外部哈希
        incremental_sha256 outer_hasher;
        outer_hasher.update(outer_key_);
        outer_hasher.update(inner_result);
        
        return outer_hasher.finalize();
    }
    
private:
    incremental_sha256 inner_hasher_;
    std::array<std::uint8_t, 64> outer_key_;
};
```

## 在 Prism 中的应用

Prism 在多个模块中使用 HMAC-SHA256，主要用于 HKDF 密钥派生和消息认证。

### HKDF 密钥派生

HMAC-SHA256 是 HKDF 的核心原语。

**提取阶段**

```cpp
// HKDF-Extract 使用 HMAC-SHA256
std::array<std::uint8_t, 32> hkdf_extract_sha256(
    std::span<const std::uint8_t> salt,
    std::span<const std::uint8_t> ikm) {
    
    // 如果 salt 为空，使用全零
    if (salt.empty()) {
        std::array<std::uint8_t, 32> zero_salt{};
        return hmac_sha256(zero_salt, ikm);
    }
    
    return hmac_sha256(salt, ikm);
}
```

**扩展阶段**

```cpp
// HKDF-Expand 使用 HMAC-SHA256
std::vector<std::uint8_t> hkdf_expand_sha256(
    std::span<const std::uint8_t, 32> prk,
    std::span<const std::uint8_t> info,
    size_t length) {
    
    constexpr size_t hash_len = 32;
    
    if (length > 255 * hash_len) {
        throw std::runtime_error("HKDF output length exceeds limit");
    }
    
    std::vector<std::uint8_t> okm;
    std::vector<std::uint8_t> t_prev;
    
    size_t n = (length + hash_len - 1) / hash_len;
    
    for (size_t i = 1; i <= n; ++i) {
        std::vector<std::uint8_t> input;
        input.insert(input.end(), t_prev.begin(), t_prev.end());
        input.insert(input.end(), info.begin(), info.end());
        input.push_back(static_cast<std::uint8_t>(i));
        
        t_prev = hmac_sha256(prk, input);
        
        size_t bytes_to_add = (i == n) 
            ? (length - okm.size()) : hash_len;
        okm.insert(okm.end(), t_prev.begin(), t_prev.begin() + bytes_to_add);
    }
    
    return okm;
}
```

**TLS 1.3 HKDF-Expand-Label**

```cpp
// TLS 1.3 HKDF-Expand-Label
std::vector<std::uint8_t> hkdf_expand_label_sha256(
    std::span<const std::uint8_t> secret,
    std::string_view label,
    std::span<const std::uint8_t> context,
    size_t length) {
    
    // 构建 HkdfLabel
    std::vector<std::uint8_t> hkdf_label;
    
    // Length (2 字节)
    hkdf_label.push_back(static_cast<std::uint8_t>(length >> 8));
    hkdf_label.push_back(static_cast<std::uint8_t>(length));
    
    // Label: "tls13 " + label
    std::string full_label = "tls13 " + std::string(label);
    hkdf_label.push_back(static_cast<std::uint8_t>(full_label.size()));
    hkdf_label.insert(hkdf_label.end(), full_label.begin(), full_label.end());
    
    // Context
    hkdf_label.push_back(static_cast<std::uint8_t>(context.size()));
    hkdf_label.insert(hkdf_label.end(), context.begin(), context.end());
    
    return hkdf_expand_sha256(secret, hkdf_label, length);
}
```

### Reality 密钥派生

Reality 协议使用 HMAC-SHA256 进行所有密钥派生。

```cpp
// Reality 握手密钥派生
session_keys derive_reality_handshake_keys(
    std::span<const std::uint8_t, 32> shared_secret,
    std::span<const std::uint8_t> handshake_hash) {
    
    session_keys keys;
    
    // 提取伪随机密钥
    auto early_secret = hkdf_extract_sha256({}, {});
    auto handshake_secret = hkdf_extract_sha256(early_secret, shared_secret);
    
    // 派生客户端密钥
    auto client_hs_secret = hkdf_expand_label_sha256(
        handshake_secret, "c hs traffic", handshake_hash, 32);
    keys.client_key = hkdf_expand_label_sha256(client_hs_secret, "key", {}, 16);
    keys.client_iv = hkdf_expand_label_sha256(client_hs_secret, "iv", {}, 12);
    
    // 派生服务端密钥
    auto server_hs_secret = hkdf_expand_label_sha256(
        handshake_secret, "s hs traffic", handshake_hash, 32);
    keys.server_key = hkdf_expand_label_sha256(server_hs_secret, "key", {}, 16);
    keys.server_iv = hkdf_expand_label_sha256(server_hs_secret, "iv", {}, 12);
    
    return keys;
}
```

### Finished 消息验证

TLS/Reality 的 Finished 消息使用 HMAC-SHA256 计算。

```cpp
// 计算 Finished 消息验证数据
std::array<std::uint8_t, 32> compute_finished_verify_data(
    std::span<const std::uint8_t, 32> finished_key,
    std::span<const std::uint8_t> transcript_hash) {
    
    // HMAC-SHA256(finished_key, transcript_hash)
    return hmac_sha256(finished_key, transcript_hash);
}

// 派生 Finished 密钥
std::array<std::uint8_t, 32> derive_finished_key(
    std::span<const std::uint8_t> traffic_secret) {
    
    return hkdf_expand_label_sha256(traffic_secret, "finished", {}, 32);
}
```

### ShadowTLS 密钥计算

ShadowTLS 使用 HMAC-SHA256 计算写入密钥。

```cpp
// ShadowTLS 写入密钥计算
std::array<std::uint8_t, 32> compute_write_key_sha256(
    std::span<const std::uint8_t> password,
    std::span<const std::uint8_t> server_random) {
    
    // SHA256(password || server_random) 然后使用 HMAC
    std::vector<std::uint8_t> input;
    input.insert(input.end(), password.begin(), password.end());
    input.insert(input.end(), server_random.begin(), server_random.end());
    
    auto base_key = sha256(input);
    
    // 进一步使用 HMAC 派生
    return hmac_sha256(base_key, std::span<const std::uint8_t>{});
}
```

### 代码位置

| 源文件 | 函数/类 | 用途 |
|--------|---------|------|
| `src/prism/crypto/hmac.cpp` | `hmac_sha256` | HMAC-SHA256 计算 |
| `src/prism/crypto/hkdf.cpp` | `hkdf_extract_sha256` | HKDF 提取 |
| `src/prism/crypto/hkdf.cpp` | `hkdf_expand_sha256` | HKDF 扩展 |
| `src/prism/crypto/hkdf.cpp` | `hkdf_expand_label` | TLS 1.3 风格派生 |
| `src/prism/stealth/reality/keygen.cpp` | `derive_handshake_keys` | Reality 密钥派生 |
| `src/prism/stealth/reality/keygen.cpp` | `compute_finished_verify_data` | Finished 验证 |

### 性能优化

**增量 HMAC 处理**

```cpp
// 增量处理大消息
class hmac_processor {
public:
    hmac_processor(std::span<const std::uint8_t> key)
        : hmac_ctx_(incremental_hmac_sha256(key)) {}
    
    void process_chunk(std::span<const std::uint8_t> chunk) {
        hmac_ctx_.update(chunk);
    }
    
    std::array<std::uint8_t, 32> finalize() {
        return hmac_ctx_.finalize();
    }
    
private:
    incremental_hmac_sha256 hmac_ctx_;
};

// 处理流数据
std::array<std::uint8_t, 32> hmac_stream(
    std::span<const std::uint8_t> key,
    std::ifstream& stream) {
    
    hmac_processor processor(key);
    std::array<std::uint8_t, 4096> buffer;
    
    while (stream.read(reinterpret_cast<char*>(buffer.data()), buffer.size())) {
        processor.process_chunk(buffer);
    }
    
    if (stream.gcount() > 0) {
        processor.process_chunk(std::span(buffer.data(), stream.gcount()));
    }
    
    return processor.finalize();
}
```

## 安全考量

### 已知安全问题

**密钥泄露**

如果密钥泄露，攻击者可以：
1. 伪造任意消息的 HMAC
2. 破解所有使用该密钥保护的通信

防御措施：
- 使用安全的密钥存储
- 定期轮换密钥
- 不同用途使用不同密钥

**弱密钥**

密钥长度不足或质量不佳会降低安全性：
- 密钥应至少 16 字节（128 位）
- 密钥应来自密码学安全随机源
- 避免使用可预测的密钥

防御措施：
- 使用 HKDF 派生密钥
- 使用安全随机源生成密钥
- 检查密钥质量

**时序攻击**

非恒定时间的 HMAC 比较可能泄露信息：
- 普通比较会在第一个不匹配字节时提前返回
- 通过时间分析可以推断 HMAC 值

防御措施：
- 使用恒定时间比较函数
- 确保所有操作在恒定时间内完成

### 安全建议

1. **密钥管理**
   - 使用足够长的密钥（推荐 32 字节）
   - 不同用途使用不同密钥
   - 定期轮换密钥
   - 使用后安全清零密钥材料

2. **实现安全**
   - 使用经过验证的加密库
   - 使用恒定时间比较函数
   - 正确处理密钥预处理

3. **输出处理**
   - 32 字节输出已足够安全
   - 截断输出会降低安全性
   - 输出可以用于认证或进一步派生

### 安全边界

| 属性 | HMAC-SHA256 | 说明 |
|------|-------------|------|
| 输出长度 | 256 位 | 32 字节 |
| 伪造攻击 | 2^256 | 理论值 |
| 密钥恢复 | 2^L | L 为密钥长度 |
| 抗长度扩展 | 天然防御 | HMAC 结构 |

### 实现清单

- [ ] 使用经过验证的库实现
- [ ] 使用足够长的密钥
- [ ] 恒定时间比较 HMAC 值
- [ ] 正确处理密钥预处理
- [ ] 不同用途使用不同密钥
- [ ] 使用后安全清零密钥
- [ ] 定期轮换密钥

## 最佳实践

### 密钥管理

**密钥生成**

```cpp
// 安全的 HMAC 密钥生成
std::array<std::uint8_t, 32> generate_hmac_key() {
    std::array<std::uint8_t, 32> key;
    RAND_bytes(key.data(), 32);  // 使用安全随机源
    return key;
}

// 从主密钥派生 HMAC 密钥
std::array<std::uint8_t, 32> derive_hmac_key(
    std::span<const std::uint8_t> master_key,
    std::string_view purpose) {
    
    // 使用 HKDF 派生
    std::vector<std::uint8_t> info;
    info.insert(info.end(), purpose.begin(), purpose.end());
    
    return hkdf_expand_sha256(master_key, info, 32);
}
```

**密钥分离**

```cpp
// 不同用途使用不同密钥
struct hmac_keys {
    std::array<std::uint8_t, 32> authentication_key;
    std::array<std::uint8_t, 32> encryption_key_derivation_key;
    std::array<std::uint8_t, 32> integrity_key;
};

hmac_keys derive_all_hmac_keys(
    std::span<const std::uint8_t> master_key) {
    
    hmac_keys keys;
    
    keys.authentication_key = derive_hmac_key(master_key, "authentication");
    keys.encryption_key_derivation_key = derive_hmac_key(master_key, "encryption");
    keys.integrity_key = derive_hmac_key(master_key, "integrity");
    
    return keys;
}
```

### 恒定时间比较

```cpp
// 恒定时间 HMAC 比较
bool verify_hmac_sha256(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> message,
    std::span<const std::uint8_t> expected_hmac) {
    
    auto computed = hmac_sha256(key, message);
    
    // 必须使用恒定时间比较
    if (computed.size() != expected_hmac.size()) {
        return false;
    }
    
    // 使用 OpenSSL 的恒定时间比较
    return CRYPTO_memcmp(computed.data(), expected_hmac.data(), computed.size()) == 0;
}

// 自己实现恒定时间比较
bool constant_time_compare(
    std::span<const std::uint8_t> a,
    std::span<const std::uint8_t> b) {
    
    if (a.size() != b.size()) {
        return false;
    }
    
    std::uint8_t result = 0;
    for (size_t i = 0; i < a.size(); ++i) {
        result |= a[i] ^ b[i];
    }
    
    return result == 0;
}
```

### 密钥轮换

```cpp
// HMAC 密钥轮换策略
class hmac_key_rotator {
public:
    hmac_key_rotator(std::chrono::hours rotation_interval = std::chrono::hours(24))
        : rotation_interval_(rotation_interval) {
        current_key_ = generate_hmac_key();
        key_created_ = std::chrono::system_clock::now();
    }
    
    std::array<std::uint8_t, 32> get_current_key() {
        check_rotation();
        return current_key_;
    }
    
    void check_rotation() {
        auto now = std::chrono::system_clock::now();
        auto age = now - key_created_;
        
        if (age > rotation_interval_) {
            rotate_key();
        }
        
        if (usage_count_ > MAX_USAGES) {
            rotate_key();
        }
    }
    
private:
    void rotate_key() {
        // 安全清理旧密钥
        OPENSSL_cleanse(current_key_.data(), 32);
        
        // 生成新密钥
        current_key_ = generate_hmac_key();
        key_created_ = std::chrono::system_clock::now();
        usage_count_ = 0;
    }
    
    std::array<std::uint8_t, 32> current_key_;
    std::chrono::system_clock::time_point key_created_;
    std::chrono::hours rotation_interval_;
    std::atomic<size_t> usage_count_{0};
    
    static constexpr size_t MAX_USAGES = 1000000;
};
```

### 错误处理

```cpp
// 安全的 HMAC 验证流程
struct hmac_result {
    bool valid;
    std::array<std::uint8_t, 32> computed;
    // 不包含详细错误信息
};

hmac_result safe_verify_hmac(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> message,
    std::span<const std::uint8_t> expected_hmac) {
    
    hmac_result result;
    result.computed = hmac_sha256(key, message);
    result.valid = constant_time_compare(result.computed, expected_hmac);
    
    // 认证失败时不泄露详细信息
    return result;
}
```

## 常见问题

### Q1: HMAC-SHA256 与 HMAC-SHA1 有什么区别？

主要区别：
- 输出长度：HMAC-SHA256 为 32 字节，HMAC-SHA1 为 20 字节
- 安全强度：HMAC-SHA256 更高
- 底层哈希：SHA-256 比 SHA-1 更安全

HMAC-SHA256 是现代应用的首选。

### Q2: HMAC-SHA256 的输出可以截断吗？

可以截断，但会降低安全性：
- 截断到 128 位：128 位安全强度
- 截断到 192 位：192 位安全强度
- 截断到 256 位：256 位安全强度（完整）

建议使用完整的 32 字节输出。

### Q3: HMAC-SHA256 密钥长度有什么要求？

HMAC-SHA256 可以接受任意长度的密钥：
- 密钥超过 64 字节会被哈希到 32 字节
- 密钥短于 64 字节会被零填充
- 推荐 32 字节密钥获得最佳安全性和效率

### Q4: HMAC-SHA256 可以用作密钥派生吗？

是的。HKDF 使用 HMAC-SHA256 作为核心原语：
- Extract: HMAC-SHA256(salt, IKM)
- Expand: HMAC-SHA256 的迭代调用

这是 TLS 1.3 的标准密钥派生方法。

### Q5: HMAC-SHA256 与 AES-GCM 的 MAC 有什么区别？

比较：
- HMAC-SHA256：独立 MAC，输出 32 字节
- AES-GCM：内置认证标签，输出 16 字节（可配置）

两者都安全，用途不同：
- HMAC-SHA256：独立认证场景
- AES-GCM：认证加密场景

### Q6: HMAC-SHA256 在 TLS 1.3 中的作用？

TLS 1.3 使用 HMAC-SHA256：
- HKDF 原语：所有密钥派生
- Finished 验证：握手完整性
- 记录层：TLS_AES_128_GCM_SHA256

HMAC-SHA256 是 TLS 1.3 的核心组件。

### Q7: HMAC-SHA256 可以防长度扩展攻击吗？

是的。HMAC 构造天然防御长度扩展攻击：
- 内部哈希的输出不直接暴露
- 外部哈希重新混合内部输出
- 攻击者无法从 HMAC 值推断内部状态

### Q8: HMAC-SHA256 的性能如何？

性能数据（单线程）：
- 纯软件：约 200-300 MB/s
- SHA-NI 加速：约 500-1000 MB/s

对于大多数应用，性能足够。

### Q9: HMAC-SHA256 与 Poly1305 如何选择？

比较：
- HMAC-SHA256：通用 MAC，广泛支持
- Poly1305：快速 MAC，与 ChaCha20 配合

选择建议：
- 独立认证场景：HMAC-SHA256
- AEAD 场景：Poly1305（ChaCha20-Poly1305）

### Q10: Reality 协议为什么使用 HMAC-SHA256？

Reality 使用 HMAC-SHA256 的原因：
- TLS 1.3 标准：与 TLS 1.3 密钥派生兼容
- 高安全性：256 位输出
- 广泛支持：所有加密库都支持
- HKDF 原语：密钥派生的标准方法