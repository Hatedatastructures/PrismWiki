---
title: "AES-GCM"
category: "crypto"
type: ref
module: ref
source: "NIST SP 800-38D"
tags: [密码学, aes, gcm, aead, 认证加密]
created: 2026-05-15
updated: 2026-05-17
layer: ref
---

# AES-GCM

**类别**: 密码学

## 概述

AES-GCM（Advanced Encryption Standard in Galois/Counter Mode）是一种基于 AES 分组密码的认证加密模式，由 NIST 在 SP 800-38D 中标准化。它同时提供数据的机密性（加密）和完整性（认证），是现代密码学中最重要的 AEAD（Authenticated Encryption with Associated Data）算法之一。

AES-GCM 的核心思想是将 AES 的 CTR（Counter）模式加密与 GHASH 乘法器结合，在一次操作中完成加密和认证。这种设计使得 AES-GCM 能够高效地处理大量数据，同时保证加密数据的完整性和真实性。在现代处理器上，AES-GCM 可以利用 AES-NI 硬件指令和 PCLMULQDQ 指令实现极高的吞吐量。

AES-GCM 支持三种 AES 密钥长度：128 位、192 位和 256 位。AES-128-GCM 提供 128 位的安全强度，AES-192-GCM 提供 192 位的安全强度，AES-256-GCM 提供 256 位的安全强度。在实际应用中，AES-128-GCM 最为常用，因为它在安全性和性能之间提供了良好的平衡。

GCM 模式的一个独特特性是支持附加认证数据（AAD）。AAD 是指不需要加密但需要认证的数据。这在网络协议中非常有用，例如在 TLS 记录层中，头部信息需要认证但不需要加密。发送方和接收方可以验证 AAD 是否被篡改，而无需将其包含在加密数据中。

AES-GCM 的加密输出包括两部分：密文和认证标签（Authentication Tag）。认证标签的长度可配置，推荐使用 128 位以获得最佳安全性。接收方在解密前首先验证认证标签，只有验证通过后才会解密密文。这种"先验证后解密"的顺序可以防止许多攻击，如填充预言机攻击。

AES-GCM 在 TLS 1.2 和 TLS 1.3 中都是强制实现的密码套件。TLS 1.3 中的 TLS_AES_128_GCM_SHA256 和 TLS_AES_256_GCM_SHA384 都使用 AES-GCM 作为加密算法。在 WireGuard VPN 协议中，AES-GCM 也是可选的加密算法之一。

AES-GCM 的性能优势主要体现在现代处理器上。Intel 和 AMD 处理器提供了 AES-NI 指令集扩展，可以加速 AES 加密操作。PCLMULQDQ 指令可以加速 GHASH 中的有限域乘法。结合这些硬件加速，AES-GCM 可以达到每秒数十吉字节的吞吐量。

然而，AES-GCM 也有一些需要注意的安全限制。最重要的是，每个密钥只能使用有限的 Nonce 值。如果使用相同的密钥和 Nonce 加密不同的消息，攻击者可以恢复认证密钥并伪造消息。因此，正确的 Nonce 管理对于 AES-GCM 的安全性至关重要。

在现代软件实现中，AES-GCM 通常使用硬件加速。在没有 AES-NI 的平台上，可以使用软件实现，但性能会显著下降。对于嵌入式系统和物联网设备，ChaCha20-Poly1305 通常是更好的选择，因为它在纯软件实现中性能更优且恒定时间。

AES-GCM 的设计还考虑了并行性。CTR 模式的加密操作可以完全并行化，GHASH 也可以通过聚合技术实现一定程度的并行化。这使得 AES-GCM 在多核处理器上可以获得接近线性的性能提升。

在代理服务器和网络协议实现中，AES-GCM 是最常用的加密算法之一。它提供了高性能和高安全性的结合，同时支持 AAD 功能，使其非常适合网络协议的数据包加密。Prism 在 Reality 协议和 TLS 记录层中使用 AES-GCM 进行数据加密和认证。

总的来说，AES-GCM 是现代密码学中最重要的认证加密算法之一。它在安全性、性能和功能之间取得了良好的平衡，是 TLS 1.3、IPsec、SSH 等协议的首选加密算法。理解 AES-GCM 的工作原理和安全限制对于安全地实现和使用它至关重要。

## 算法原理

### AES 分组密码

AES（Advanced Encryption Standard）是由比利时密码学家 Joan Daemen 和 Vincent Rijmen 设计的分组密码，于 2001 年被 NIST 选为新的加密标准。AES 使用 SPN（Substitution-Permutation Network）结构，处理 128 位（16 字节）的分组。

**密钥扩展**

AES 支持 128 位、192 位和 256 位三种密钥长度。密钥扩展过程将原始密钥扩展为轮密钥：

| 密钥长度 | 轮数 | 扩展密钥长度（字节） |
|----------|------|---------------------|
| 128 位 | 10 | 176 |
| 192 位 | 12 | 208 |
| 256 位 | 14 | 240 |

密钥扩展算法使用以下操作：
- S-Box 替换（SubWord）
- 字旋转（RotWord）
- 轮常量异或（Rcon）

**轮函数**

AES 的每一轮（除最后一轮外）包含四个操作：

1. **SubBytes（字节替换）**：使用 S-Box 对每个字节进行非线性替换。S-Box 是一个 256 字节的查找表，通过在 GF(2^8) 上的乘法逆元后进行仿射变换构造。

2. **ShiftRows（行移位）**：对状态矩阵的行进行循环左移：
   - 第 0 行：不移位
   - 第 1 行：左移 1 字节
   - 第 2 行：左移 2 字节
   - 第 3 行：左移 3 字节

3. **MixColumns（列混合）**：对状态矩阵的每列进行矩阵乘法。矩阵乘法在 GF(2^8) 中进行，使用的不可约多项式为 x^8 + x^4 + x^3 + x + 1。

4. **AddRoundKey（轮密钥加）**：将状态矩阵与轮密钥进行异或。

最后一轮省略 MixColumns 操作。

**加密流程**

```
明文 P (128 位)
    ↓
AddRoundKey(P, K0)
    ↓
Round 1: SubBytes → ShiftRows → MixColumns → AddRoundKey
Round 2: SubBytes → ShiftRows → MixColumns → AddRoundKey
    ...
Round N-1: SubBytes → ShiftRows → MixColumns → AddRoundKey
Round N: SubBytes → ShiftRows → AddRoundKey
    ↓
密文 C (128 位)
```

### GCM 模式结构

GCM（Galois/Counter Mode）结合了 CTR 模式加密和 GHASH 乘法器认证。GCM 的核心组件包括：

**CTR 模式加密**

CTR（Counter）模式将分组密码转换为流密码：

```
Ci = Pi ⊕ AES_K(Counter_i)
```

其中：
- Ci 是第 i 个密文块
- Pi 是第 i 个明文块
- AES_K 是使用密钥 K 的 AES 加密
- Counter_i 是第 i 个计数器值

计数器从初始值 ICB（Initial Counter Block）开始递增：
- Counter_0 = ICB
- Counter_1 = ICB + 1
- Counter_2 = ICB + 2
- ...

**GHASH 乘法器**

GHASH 是在 GF(2^128) 上进行的乘法运算，用于计算认证标签。使用的不可约多项式为：

```
G = x^128 + x^7 + x^2 + x + 1
```

GHASH 的计算过程：

```
GHASH(H, A, C) = A1·H^m+n+1 + A2·H^m+n + ... + Am·H^n+1
              + C1·H^n + C2·H^n-1 + ... + Cn·H + L·H^0
```

其中：
- H = AES_K(0^128)，哈希子密钥
- A = AAD 数据块 A1, A2, ..., Am
- C = 密文数据块 C1, C2, ..., Cn
- L = len(A) || len(C)，长度块

### AES-GCM 完整算法

**输入**：
- 密钥 K（128/192/256 位）
- Nonce IV（建议 96 位）
- 明文 P
- 附加认证数据 AAD

**输出**：
- 密文 C
- 认证标签 T（推荐 128 位）

**加密步骤**：

1. **生成哈希子密钥**：
   ```
   H = AES_K(0^128)
   ```

2. **生成初始计数器块**：
   ```
   如果 len(IV) = 96 位：
       J0 = IV || 0^31 || 1
   否则：
       J0 = GHASH(H, {}, IV)
   ```

3. **CTR 模式加密**：
   ```
   对于 i = 1, 2, ..., n：
       Counter_i = J0 + i
       C_i = P_i ⊕ AES_K(Counter_i)
   ```

4. **计算认证标签**：
   ```
   S = GHASH(H, AAD, C)
   T = S ⊕ AES_K(J0)
   ```

**解密验证步骤**：

1. **重新计算认证标签**：
   ```
   S' = GHASH(H, AAD, C)
   T' = S' ⊕ AES_K(J0)
   ```

2. **恒定时间比较**：
   ```
   如果 T' = T：
       解密成功，返回明文 P
   否则：
       认证失败，拒绝解密
   ```

### 数学基础

**有限域 GF(2^128)**

GCM 在有限域 GF(2^128) 上进行运算。域元素表示为 128 位多项式，系数为 0 或 1。

乘法运算：
```
(a(x) · b(x)) mod G
```

其中 G = x^128 + x^7 + x^2 + x + 1 是不可约多项式。

**PCLMULQDQ 加速**

Intel 的 PCLMULQDQ 指令可以高效执行 64 位无进位乘法。通过适当的算法，可以用多次 PCLMULQDQ 实现 128 位乘法和约简。

### 安全性分析

**安全强度**

| 密钥长度 | 安全强度 | 推荐用途 |
|----------|----------|----------|
| 128 位 | 128 位 | 通用加密 |
| 192 位 | 192 位 | 高安全需求 |
| 256 位 | 256 位 | 最高安全需求 |

**Nonce 管理的重要性**

AES-GCM 最关键的安全限制是 Nonce 的唯一性。使用相同密钥和 Nonce 加密两条不同消息会导致灾难性的安全后果：

1. **认证密钥泄露**：
   如果 CTR 模式的计数器值重复，攻击者可以计算：
   ```
   AES_K(J0 + 1) = C1 ⊕ P1 = C1' ⊕ P1'
   ```
   这泄露了加密密钥流，攻击者可以解密消息。

2. **认证标签伪造**：
   攻击者可以恢复 H 值并伪造任意消息的认证标签。

**Nonce 使用建议**：
- 使用 96 位 Nonce（IV），与计数器拼接形成 128 位初始计数器
- 每个密钥的生命周期内，Nonce 不应重复
- 推荐使用计数器作为 Nonce，每个消息递增
- 随机 Nonce 需要限制消息数量（对于 96 位随机 Nonce，建议不超过 2^32 条消息）

**认证标签长度**

认证标签可以截断，但会降低安全性：

| 标签长度 | 安全强度 | 说明 |
|----------|----------|------|
| 128 位 | 128 位 | 推荐用于高安全性 |
| 120 位 | 120 位 | 可接受 |
| 96 位 | 96 位 | 可接受，性能略优 |
| 64 位 | 64 位 | 不推荐，易受伪造攻击 |
| 32 位 | 32 位 | 不安全 |

## 在 Prism 中的应用

Prism 在多个模块中使用 AES-GCM，主要用于 Reality 协议和 TLS 记录层的加密认证。

### Reality 协议实现

Reality 是一种 TLS 伪装协议，使用 AES-GCM 保护通信内容。Prism 的 Reality 实现位于 `stealth/reality` 模块。

**加密 TLS 记录**

在 Reality 协议中，服务端使用 AES-GCM 加密 TLS 记录，使流量看起来像正常的 TLS 通信：

```cpp
// 加密 TLS 记录
auto encrypt_tls_record(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> nonce,
    std::span<const std::uint8_t> plaintext,
    std::span<const std::uint8_t> aad) -> std::vector<std::uint8_t> {
    
    // 初始化 AES-GCM 上下文
    aead_context ctx;
    ctx.init(key, aead_algorithm::aes_128_gcm);
    
    // 加密并生成认证标签
    auto [ciphertext, tag] = ctx.seal(nonce, plaintext, aad);
    
    // 组合输出：nonce || ciphertext || tag
    std::vector<std::uint8_t> output;
    output.reserve(nonce.size() + ciphertext.size() + tag.size());
    output.insert(output.end(), nonce.begin(), nonce.end());
    output.insert(output.end(), ciphertext.begin(), ciphertext.end());
    output.insert(output.end(), tag.begin(), tag.end());
    
    return output;
}
```

**解密验证**

Reality 客户端使用 AES-GCM 解密并验证服务端发送的记录：

```cpp
// 解密 TLS 记录
auto decrypt_tls_record(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> nonce,
    std::span<const std::uint8_t> ciphertext,
    std::span<const std::uint8_t> tag,
    std::span<const std::uint8_t> aad) -> std::optional<std::vector<std::uint8_t>> {
    
    aead_context ctx;
    ctx.init(key, aead_algorithm::aes_128_gcm);
    
    // 验证并解密
    auto plaintext = ctx.open(nonce, ciphertext, tag, aad);
    if (!plaintext) {
        // 认证失败
        return std::nullopt;
    }
    
    return plaintext;
}
```

### AEAD 抽象层

Prism 提供统一的 AEAD 抽象层，支持多种算法（AES-GCM、ChaCha20-Poly1305 等）：

**AEAD 上下文初始化**

```cpp
// src/prism/crypto/aead.cpp
enum class aead_algorithm {
    aes_128_gcm,
    aes_256_gcm,
    chacha20_poly1305,
    // ...
};

struct aead_context {
    void init(std::span<const std::uint8_t> key, aead_algorithm algo) {
        switch (algo) {
            case aead_algorithm::aes_128_gcm:
                cipher_ = EVP_aes_128_gcm();
                break;
            case aead_algorithm::aes_256_gcm:
                cipher_ = EVP_aes_256_gcm();
                break;
            case aead_algorithm::chacha20_poly1305:
                cipher_ = EVP_chacha20_poly1305();
                break;
        }
        // 初始化 EVP 上下文
        EVP_CIPHER_CTX_init(ctx_);
        EVP_EncryptInit_ex(ctx_, cipher_, nullptr, key.data(), nullptr);
    }
    
    auto seal(
        std::span<const std::uint8_t> nonce,
        std::span<const std::uint8_t> plaintext,
        std::span<const std::uint8_t> aad) -> seal_result {
        
        // 设置 nonce
        EVP_CIPHER_CTX_ctrl(ctx_, EVP_CTRL_GCM_SET_IV, nonce.size(), nonce.data());
        
        // 添加 AAD
        int len;
        EVP_EncryptUpdate(ctx_, nullptr, &len, aad.data(), aad.size());
        
        // 加密
        std::vector<std::uint8_t> ciphertext(plaintext.size());
        EVP_EncryptUpdate(ctx_, ciphertext.data(), &len, 
                          plaintext.data(), plaintext.size());
        
        // 获取认证标签
        std::array<std::uint8_t, 16> tag;
        EVP_CIPHER_CTX_ctrl(ctx_, EVP_CTRL_GCM_GET_TAG, 16, tag.data());
        
        return {ciphertext, tag};
    }
};
```

### 密钥派生

AES-GCM 的密钥通过 HKDF 从共享密钥派生：

```cpp
// 从 ECDHE 共享密钥派生 AES-GCM 密钥
auto derive_aes_gcm_key(
    std::span<const std::uint8_t> shared_secret,
    std::span<const std::uint8_t> label) -> std::array<std::uint8_t, 16> {
    
    // 使用 HKDF-Expand-Label 派生密钥
    return hkdf_expand_label(shared_secret, label, {}, 16);
}
```

### 代码位置

| 源文件 | 函数/类 | 用途 |
|--------|---------|------|
| `src/prism/crypto/aead.cpp` | `aead_context` | AEAD 抽象层 |
| `src/prism/stealth/reality/response.cpp` | `encrypt_tls_record` | Reality 记录加密 |
| `src/prism/stealth/reality/seal.cpp` | `seal` | Reality 传输层加密 |
| `src/prism/stealth/reality/keygen.cpp` | `derive_handshake_keys` | 密钥派生 |

### 性能优化

Prism 利用现代处理器的硬件加速：

1. **AES-NI 指令集**：加速 AES 加密
2. **PCLMULQDQ 指令**：加速 GHASH 计算
3. **AVX-512 指令**：并行处理多个数据块

```cpp
// 检查硬件支持
bool has_aes_ni() {
    return (cpuid.ecx & (1 << 25)) != 0;  // AES-NI
}

bool has_pclmulqdq() {
    return (cpuid.ecx & (1 << 1)) != 0;   // PCLMULQDQ
}
```

## 安全考量

### 已知安全问题

**Nonce 重用攻击**

这是 AES-GCM 最严重的安全问题。使用相同的密钥和 Nonce 加密两条不同消息会导致：
1. 加密密钥流泄露
2. 认证密钥 H 恢复
3. 消息伪造成为可能

防御措施：
- 使用计数器作为 Nonce，确保单调递增
- 每个会话使用不同的密钥
- 限制每个密钥加密的消息数量

**短标签攻击**

截断认证标签降低安全性。64 位标签在 2^32 次尝试后可能被伪造。

防御措施：
- 使用完整的 128 位标签
- 对于短标签，实现重试计数器限制

**时序攻击**

非恒定时间的 GHASH 实现可能泄露信息。

防御措施：
- 使用恒定时间实现
- 利用硬件加速（PCLMULQDQ）

**弱密钥**

虽然 AES 没有已知的弱密钥，但密钥生成必须使用密码学安全的随机数生成器。

### 安全建议

1. **Nonce 管理**
   - 优先使用 96 位 Nonce
   - 每个密钥的生命周期内保持 Nonce 唯一
   - 实现自动 Nonce 递增机制

2. **密钥管理**
   - 使用 HKDF 派生密钥，确保不同用途使用不同密钥
   - 定期轮换密钥
   - 使用安全的密钥存储

3. **标签验证**
   - 始终使用完整的 128 位标签
   - 在解密前验证标签
   - 使用恒定时间比较

4. **AAD 使用**
   - 包含所有需要认证但不需要加密的数据
   - 包含协议版本、消息长度等元数据

### 安全边界

| 攻击类型 | 条件 | 安全性 |
|----------|------|--------|
| 明文恢复 | Nonce 唯一 | 2^128 |
| 标签伪造 | Nonce 唯一 | 2^128 |
| Nonce 重用 | Nonce 相同 | 破坏 |

### 实现清单

- [ ] Nonce 必须唯一
- [ ] 使用 128 位认证标签
- [ ] 先验证后解密
- [ ] 恒定时间比较标签
- [ ] 包含相关 AAD
- [ ] 密钥派生使用 HKDF
- [ ] 实现密钥轮换机制

## 最佳实践

### Nonce 管理

**推荐做法：计数器模式**

```cpp
class gcm_nonce_manager {
public:
    gcm_nonce_manager() : counter_(0) {}
    
    std::array<std::uint8_t, 12> next_nonce() {
        std::array<std::uint8_t, 12> nonce{};
        
        // 高位固定，低位递增
        std::uint64_t current = counter_.fetch_add(1);
        
        // 检查溢出
        if (current >= (1ULL << 32)) {
            throw std::runtime_error("Nonce space exhausted");
        }
        
        // 写入 nonce（大端序）
        for (int i = 0; i < 8; ++i) {
            nonce[11 - i] = static_cast<std::uint8_t>(current >> (i * 8));
        }
        
        return nonce;
    }
    
private:
    std::atomic<std::uint64_t> counter_;
};
```

**随机 Nonce 的限制**

```cpp
// 对于 96 位随机 Nonce，限制消息数量
constexpr std::uint64_t MAX_MESSAGES_WITH_RANDOM_NONCE = (1ULL << 32);

bool should_rotate_key(std::uint64_t messages_encrypted) {
    return messages_encrypted >= MAX_MESSAGES_WITH_RANDOM_NONCE;
}
```

### 密钥管理

**密钥派生**

```cpp
// 从主密钥派生 AES-GCM 密钥
auto derive_gcm_key(
    std::span<const std::uint8_t> master_key,
    std::string_view context) -> std::array<std::uint8_t, 16> {
    
    // 使用 HKDF 派生
    std::vector<std::uint8_t> info(context.begin(), context.end());
    info.push_back(0x01);  // 密钥用途标识
    
    return hkdf_expand(master_key, info, 16);
}
```

**密钥轮换**

```cpp
class key_rotator {
public:
    void rotate_if_needed() {
        auto now = std::chrono::system_clock::now();
        auto age = now - current_key_.created_at;
        
        // 基于时间轮换（例如 24 小时）
        if (age > std::chrono::hours(24)) {
            rotate_key();
        }
        
        // 基于使用量轮换
        if (current_key_.messages_encrypted > MESSAGE_LIMIT) {
            rotate_key();
        }
    }
    
private:
    static constexpr std::uint64_t MESSAGE_LIMIT = (1ULL << 32);
    struct key_info {
        std::array<std::uint8_t, 16> key;
        std::chrono::system_clock::time_point created_at;
        std::atomic<std::uint64_t> messages_encrypted{0};
    };
    key_info current_key_;
};
```

### AAD 使用

**包含协议元数据**

```cpp
struct tls_record_header {
    std::uint8_t content_type;
    std::uint16_t version;
    std::uint16_t length;
};

auto build_aad(const tls_record_header& header) -> std::vector<std::uint8_t> {
    std::vector<std::uint8_t> aad(5);
    aad[0] = header.content_type;
    aad[1] = (header.version >> 8) & 0xFF;
    aad[2] = header.version & 0xFF;
    aad[3] = (header.length >> 8) & 0xFF;
    aad[4] = header.length & 0xFF;
    return aad;
}
```

### 错误处理

**安全解密流程**

```cpp
auto secure_decrypt(
    const aead_context& ctx,
    std::span<const std::uint8_t> nonce,
    std::span<const std::uint8_t> ciphertext,
    std::span<const std::uint8_t> tag,
    std::span<const std::uint8_t> aad) -> std::optional<std::vector<std::uint8_t>> {
    
    // 验证标签（恒定时间）
    auto result = ctx.open(nonce, ciphertext, tag, aad);
    
    if (!result) {
        // 认证失败，不返回任何信息
        // 注意：不要记录详细的失败原因
        return std::nullopt;
    }
    
    return result;
}
```

### 性能优化

**批量处理**

```cpp
// 使用 AES-NI 和 PCLMULQDQ 优化
void process_batch(
    std::span<const message> messages,
    std::span<std::vector<std::uint8_t>> outputs) {
    
    for (size_t i = 0; i < messages.size(); ++i) {
        outputs[i] = encrypt_single(messages[i]);
    }
}
```

**避免小数据包**

```cpp
// 对于小数据包，考虑使用 ChaCha20-Poly1305
aead_algorithm select_algorithm(size_t payload_size, bool has_aes_ni) {
    if (!has_aes_ni || payload_size < 64) {
        return aead_algorithm::chacha20_poly1305;
    }
    return aead_algorithm::aes_128_gcm;
}
```

## 常见问题

### Q1: AES-GCM 的 Nonce 可以重复使用吗？

绝对不可以。使用相同的密钥和 Nonce 加密两条不同消息会导致灾难性的安全后果：
1. 攻击者可以计算两条消息的异或
2. 攻击者可以恢复认证密钥 H
3. 攻击者可以伪造任意消息

每个密钥的生命周期内，Nonce 必须唯一。

### Q2: 推荐的 Nonce 长度是多少？

推荐使用 96 位（12 字节）Nonce。这个长度允许简单地将 Nonce 与 32 位计数器拼接形成 128 位初始计数器块。

如果使用随机 Nonce，建议限制每个密钥加密的消息数量不超过 2^32 条，以避免 Nonce 碰撞。

### Q3: AES-128-GCM 和 AES-256-GCM 应该选择哪个？

AES-128-GCM 提供 128 位安全强度，对于绝大多数应用已经足够。AES-256-GCM 提供 256 位安全强度，适用于：
1. 需要长期保密的数据
2. 对量子计算攻击有顾虑的场景
3. 法规要求使用更长的密钥

AES-128-GCM 在某些场景下性能更好（轮数更少）。

### Q4: 认证标签可以截断吗？

可以截断，但不推荐。截断标签会降低安全性：
- 128 位标签：128 位安全强度
- 96 位标签：96 位安全强度
- 64 位标签：64 位安全强度（不推荐）
- 32 位标签：不安全

如果必须截断，建议使用至少 96 位标签。

### Q5: AAD 是必须的吗？

AAD 不是必须的，但强烈推荐使用。AAD 允许包含需要认证但不需要加密的数据，例如：
- 协议头
- 消息长度
- 序列号
- 版本信息

正确使用 AAD 可以防止各种协议攻击。

### Q6: AES-GCM 与 ChaCha20-Poly1305 如何选择？

考虑以下因素：

| 因素 | AES-GCM | ChaCha20-Poly1305 |
|------|---------|-------------------|
| 硬件加速 | AES-NI, PCLMULQDQ | 无需特定指令 |
| 纯软件性能 | 较慢 | 较快 |
| 恒定时间 | 需要小心实现 | 天然恒定时间 |
| 移动设备 | 取决于 CPU | 通常更快 |
| 安全性 | 成熟 | 成熟 |

如果目标平台支持 AES-NI，AES-GCM 通常是更好的选择。对于嵌入式设备或不支持 AES-NI 的平台，ChaCha20-Poly1305 更合适。

### Q7: 如何处理认证失败？

认证失败意味着密文或 AAD 被篡改。正确的处理方式：
1. 不要尝试解密
2. 不泄露具体失败原因
3. 记录安全日志（不含敏感信息）
4. 终止连接或请求重传

### Q8: AES-GCM 可以用于流加密吗？

AES-GCM 本质上是分组加密模式，但可以处理任意长度的数据（通过 CTR 模式）。对于非常大的数据流，建议：
1. 将数据分成多个记录
2. 每个记录使用独立的 Nonce
3. 定期重新协商密钥

### Q9: AES-GCM 的性能如何？

在现代处理器上（支持 AES-NI 和 PCLMULQDQ）：
- AES-128-GCM：约 10-20 GB/s
- AES-256-GCM：约 8-15 GB/s

在没有硬件加速的情况下：
- AES-128-GCM：约 50-200 MB/s
- AES-256-GCM：约 40-150 MB/s

### Q10: TLS 1.3 为什么强制要求 AES-GCM？

TLS 1.3 强制要求支持 AES-128-GCM 和 AES-256-GCM，原因包括：
1. 高安全性：经过充分分析，没有已知的实际攻击
2. 高性能：现代 CPU 支持硬件加速
3. 广泛部署：已有大量实现和部署经验
4. 功能完整：支持 AAD，适合 TLS 记录层

同时，TLS 1.3 也要求支持 ChaCha20-Poly1305，为不支持 AES-NI 的平台提供高性能替代方案。