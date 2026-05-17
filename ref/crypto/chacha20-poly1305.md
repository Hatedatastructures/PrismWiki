---
title: "ChaCha20-Poly1305"
category: "crypto"
type: ref
module: ref
source: "RFC 8439"
tags: [密码学, chacha20, poly1305, aead, 认证加密]
created: 2026-05-15
updated: 2026-05-17
---

# ChaCha20-Poly1305

**类别**: 密码学

## 概述

ChaCha20-Poly1305 是一种现代 AEAD（Authenticated Encryption with Associated Data）算法，由 Daniel J. Bernstein 设计。它结合了 ChaCha20 流密码进行加密和 Poly1305 消息认证码进行认证，在单次操作中同时提供数据的机密性和完整性保护。

ChaCha20-Poly1305 的设计理念是在纯软件实现中获得高性能，同时保持恒定时间执行以防止时序攻击。与 AES-GCM 不同，ChaCha20-Poly1305 不依赖任何 CPU 特定的硬件加速指令，这使得它在各种平台上都能保持一致的性能和安全性。这一特性使得 ChaCha20-Poly1305 成为移动设备、嵌入式系统和 IoT 设备的理想选择。

ChaCha20 是 Salsa20 流密码的改进版本，由 Bernstein 在 2008 年设计。Salsa20 和 ChaCha20 都属于 eSTREAM 项目的最终候选算法，经过密码学社区的广泛分析和验证。ChaCha20 通过改进核心函数的扩散特性，在保持 Salsa20 安全性的同时略微提高了性能。

Poly1305 是一种快速的消息认证码，同样由 Bernstein 设计。它基于有限域上的多项式求值，生成 128 位（16 字节）的认证标签。Poly1305 的设计强调简单性和恒定时间执行，使其在软件实现中既高效又安全。

ChaCha20-Poly1305 在 TLS 1.2 和 TLS 1.3 中都是可选但广泛支持的密码套件。TLS 1.3 中的 TLS_CHACHA20_POLY1305_SHA256 密码套件使用 ChaCha20-Poly1305 作为加密算法。在 Google 的许多服务（如 Chrome、Google 服务器）中，ChaCha20-Poly1305 是首选的加密算法，因为它在非 AES-NI 平台上性能更好。

ChaCha20-Poly1305 的一个重要优势是恒定时间执行。AES-GCM 的 GHASH 部分需要特别小心地实现以避免时序攻击，而 ChaCha20 和 Poly1305 的所有操作天然是恒定时间的。这意味着即使攻击者能够精确测量执行时间，也无法从中推断出密钥或明文信息。

在安全性方面，ChaCha20-Poly1305 提供 256 位密钥和 128 位认证标签。加密部分提供 256 位的安全强度（虽然实际有效强度可能略低），认证部分提供 128 位的安全强度。这比 AES-128-GCM 的 128 位加密强度更高，与 AES-256-GCM 相当。

ChaCha20-Poly1305 的 Nonce 管理比 AES-GCM 更严格。它使用固定的 96 位（12 字节）Nonce，每个密钥的生命周期内 Nonce 必须唯一。与 AES-GCM 类似，Nonce 重用会导致灾难性的安全后果，包括认证密钥泄露和消息伪造。

在 TLS 1.3 的密码套件设计中，ChaCha20-Poly1305 的地位与 AES-GCM 相当。TLS 1.3 规定了三种 AEAD 密码套件：TLS_AES_128_GCM_SHA256、TLS_AES_256_GCM_SHA384 和 TLS_CHACHA20_POLY1305_SHA256。这三种都是可选的，但至少需要实现其中一种。大多数现代 TLS 实现都支持这三种。

Google 在推广 ChaCha20-Poly1305 方面起到了重要作用。Chrome 浏览器在没有 AES-NI 的平台上优先选择 ChaCha20-Poly1305。Google 服务器也广泛部署了 ChaCha20-Poly1305。这种推广使得 ChaCha20-Poly1305 在实际应用中获得了广泛的经验和验证。

ChaCha20-Poly1305 也被用于其他协议和系统。WireGuard VPN 协议使用 ChaCha20-Poly1305 作为加密算法。SSH 协议支持 chacha20-poly1305@openssh.com 密码套件。Age 文件加密工具默认使用 ChaCha20-Poly1305。

在代理服务器实现中，ChaCha20-Poly1305 常用于不支持 AES-NI 的平台。Prism 在 AEAD 抽象层中支持 ChaCha20-Poly1305，并根据平台特性自动选择最优算法。对于有 AES-NI 的 x86 平台，AES-GCM 可能更快；对于 ARM 平台或没有 AES-NI 的 x86 平台，ChaCha20-Poly1305 通常更快。

总的来说，ChaCha20-Poly1305 是一种设计精良的 AEAD 算法，在安全性、性能和实现简单性之间取得了优秀的平衡。它的恒定时间特性使其在安全性方面优于 AES-GCM，纯软件高效实现使其在各种平台上都能提供一致的性能。理解 ChaCha20-Poly1305 的工作原理对于正确使用和实现它至关重要。

## 算法原理

### ChaCha20 流密码

ChaCha20 是一种基于 ARX（Add-Rotate-Xor）结构的流密码。它的核心是一个 4x4 的 32 位整数矩阵（称为状态矩阵），通过 20 轮的加法、旋转和异或操作来更新状态。

**状态矩阵结构**

ChaCha20 的状态矩阵包含 16 个 32 位整数，排列如下：

```
       0  1  2  3
       4  5  6  7
       8  9 10 11
      12 13 14 15
```

初始化时，矩阵被填充为：

```
| const0  const1  const2  const3 |   常量 "expand 32-byte k"
|   k0      k1      k2      k3   |   密钥前 16 字节
|   k4      k5      k6      k7   |   密钥后 16 字节
|   c0      c1      n0      n1   |   计数器 + Nonce
```

其中：
- 常量：const0-3 = "expand 32-byte k" 的 ASCII 码（0x61707865, 0x3320646e, 0x79622d32, 0x6b206574）
- 密钥：k0-7 = 256 位密钥的 8 个 32 位字（小端序）
- 计数器：c0 = 32 位计数器，c1 = 0（或扩展计数器的高位）
- Nonce：n0-1 = 96 位 Nonce 的 3 个 32 位字（RFC 8439 使用 c0, c1 = 0, n0-2 = Nonce）

**Quarter Round（四分之一轮）**

ChaCha20 的核心操作是 quarter round，对状态矩阵的四个元素进行更新：

```
quarter_round(a, b, c, d):
    a += b; d ^= a; d <<<= 16;
    c += d; b ^= c; b <<<= 12;
    a += b; d ^= a; d <<<= 8;
    c += d; b ^= c; b <<<= 7;
```

其中：
- += 是模 2^32 加法
- ^= 是异或
- <<< 是循环左移

**双轮结构**

ChaCha20 执行 10 个双轮（共 20 轮），每个双轮包含：
1. 列轮（Column Round）：对每列执行 quarter round
2. 对角轮（Diagonal Round）：对对角线执行 quarter round

列轮操作的元素：
```
QR(0, 4, 8, 12)  QR(1, 5, 9, 13)  QR(2, 6, 10, 14)  QR(3, 7, 11, 15)
```

对角轮操作的元素：
```
QR(0, 5, 10, 15)  QR(1, 6, 11, 12)  QR(2, 7, 8, 13)  QR(3, 4, 9, 14)
```

**密钥流生成**

完成 20 轮后，将工作状态加到初始状态上：

```
for i in 0..16:
    output[i] = initial_state[i] + working_state[i]
```

这产生 64 字节（512 位）的密钥流。对于更长的消息，计数器递增并继续生成密钥流块。

**加密操作**

加密是简单的异或操作：

```
ciphertext = plaintext XOR keystream
```

### Poly1305 消息认证码

Poly1305 是一种基于多项式求值的消息认证码，在有限域 GF(2^130-5) 上进行计算。

**有限域 GF(2^130-5)**

Poly1305 使用的素数是 p = 2^130 - 5。域元素表示为 130 位整数，运算在模 p 下进行。

选择 p = 2^130 - 5 的原因：
1. 接近 2^130，便于实现
2. 5 是小素数，约简简单
3. 130 位足够大，防止碰撞

**认证密钥**

Poly1305 使用两个 128 位（16 字节）的密钥：
- r：用于多项式系数
- s：用于最终输出

r 有特定的位限制（clamping）：
- 清除第 0-3 位（确保 r 可被 16 整除）
- 清除第 124-127 位
- 清除第 4-7 位中的某些位

这些限制防止某些攻击并简化实现。

**多项式求值**

Poly1305 计算以下多项式：

```
auth = (c1 * r^n + c2 * r^(n-1) + ... + cn * r + s) mod (2^130-5)
```

其中：
- c1, c2, ..., cn 是消息块（每块 16 字节，最后一个块可能短于 16 字节）
- n 是块数量
- 每个块 ci 在高位附加一个 1 字节（表示块长度）

**具体计算步骤**

1. 将消息分成 16 字节的块
2. 对于每个块，在高位附加 1 字节：
   - 如果块是完整的 16 字节：附加 0x01
   - 如果块不完整：附加 0x01 到数据后面，然后用 0 填充到 17 字节
3. 计算多项式：累加器从 0 开始，对每个块计算 `accumulator = (accumulator + block) * r mod p`
4. 最后加上 s：`auth = (accumulator + s) mod 2^128`
5. 输出 16 字节认证标签

### ChaCha20-Poly1305 AEAD 构造

RFC 8439 定义了 ChaCha20-Poly1305 的 AEAD 构造。

**输入**：
- 256 位密钥 K
- 96 位 Nonce N
- 明文 P
- 附加认证数据 AAD

**输出**：
- 密文 C
- 128 位认证标签 T

**构造步骤**：

1. **生成 Poly1305 密钥**：
   ```
   poly_key = ChaCha20(K, {0,0,0,0} || N, 0..31)
   ```
   使用 ChaCha20 的第一个 32 字节输出作为 Poly1305 的 r 和 s 密钥。

2. **加密明文**：
   ```
   C = ChaCha20(K, {0,0,0,1} || N, P)
   ```
   计数器从 1 开始（而不是 0），加密明文。

3. **计算认证标签**：
   ```
   T = Poly1305(poly_key, AAD || pad16(AAD) || C || pad16(C) || len(AAD) || len(C))
   ```
   其中：
   - pad16(x) = x 如果 len(x) 是 16 的倍数，否则 x || 0^{16 - len(x) mod 16}
   - len(AAD) 和 len(C) 是 8 字节小端序整数

4. **输出格式**：
   ```
   output = C || T
   ```

**解密步骤**：

1. 生成 Poly1305 密钥（与加密相同）
2. 计算认证标签 T'
3. 恒定时间比较 T' 和 T
4. 如果匹配，解密 C；否则返回错误

### 数学细节

**ChaCha20 循环移位**

循环移位操作 <<< 在 32 位整数上进行：

```
x <<< n = (x << n) | (x >> (32 - n))
```

现代处理器通常有专门的指令（如 x86 的 ROL），但软件实现使用上述公式。

**Poly1305 模约简**

约简模 2^130-5 的算法：

```
// 假设有一个 130 位值 h
h_mod_p = h mod (2^130 - 5)

// 算法：
carry = h >> 130          // 提取高于 130 位的部分
h = h & (2^130 - 1)       // 保留 130 位
h += 5 * carry            // 加上 5 * overflow
// 可能需要再次约简
```

这个约简可以在 64 位或 128 位整数上高效实现。

### 安全性分析

**ChaCha20 安全性**

ChaCha20 基于 Salsa20，后者经过 eSTREAM 项目的详细分析：
- 最佳已知攻击约 2^250（远高于实际需求）
- 没有已知的实际攻击
- 20 轮提供足够的安全边界（核心函数的扩散在 4-5 轨迹后饱和）

**Poly1305 安全性**

Poly1305 的安全性基于：
- 多项式求值的唯一性
- r 的位限制防止某些攻击
- 128 位输出提供 128 位认证强度

**AEAD 构造安全性**

ChaCha20-Poly1305 的 AEAD 构造安全性：
- 加密强度：256 位
- 认证强度：128 位
- Nonce 重用：灾难性后果

## 在 Prism 中的应用

Prism 在 AEAD 抽象层中支持 ChaCha20-Poly1305，主要用于不支持 AES-NI 的平台或需要恒定时间保证的场景。

### AEAD 抽象层

Prism 提供统一的 AEAD 抽象层，ChaCha20-Poly1305 作为可选算法之一：

```cpp
// AEAD 算法选择
enum class aead_algorithm {
    aes_128_gcm,
    aes_256_gcm,
    chacha20_poly1305,
};

// 根据平台特性选择最优算法
aead_algorithm select_optimal_algorithm() {
    bool has_aes_ni = check_aes_ni_support();
    
    if (has_aes_ni) {
        return aead_algorithm::aes_128_gcm;  // AES-GCM 更快
    }
    
    return aead_algorithm::chacha20_poly1305;  // ChaCha 更快
}
```

### 加密操作

**初始化和加密**

```cpp
// ChaCha20-Poly1305 加密示例
auto chacha_encrypt(
    std::span<const std::uint8_t> key,      // 32 字节
    std::span<const std::uint8_t> nonce,    // 12 字节
    std::span<const std::uint8_t> plaintext,
    std::span<const std::uint8_t> aad) -> chacha_result {
    
    // 初始化 EVP 上下文
    EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
    EVP_EncryptInit_ex(ctx, EVP_chacha20_poly1305(), nullptr, 
                       key.data(), nonce.data());
    
    // 设置 AAD
    int len;
    EVP_EncryptUpdate(ctx, nullptr, &len, aad.data(), aad.size());
    
    // 加密
    std::vector<std::uint8_t> ciphertext(plaintext.size());
    EVP_EncryptUpdate(ctx, ciphertext.data(), &len, 
                      plaintext.data(), plaintext.size());
    
    // 获取认证标签
    std::array<std::uint8_t, 16> tag;
    EVP_EncryptFinal_ex(ctx, nullptr, &len);
    EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_AEAD_GET_TAG, 16, tag.data());
    
    EVP_CIPHER_CTX_free(ctx);
    
    return {ciphertext, tag};
}
```

### 解密验证

**安全解密流程**

```cpp
// ChaCha20-Poly1305 解密示例
auto chacha_decrypt(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> nonce,
    std::span<const std::uint8_t> ciphertext,
    std::span<const std::uint8_t> tag,
    std::span<const std::uint8_t> aad) -> std::optional<std::vector<std::uint8_t>> {
    
    EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
    EVP_DecryptInit_ex(ctx, EVP_chacha20_poly1305(), nullptr, 
                       key.data(), nonce.data());
    
    // 设置 AAD
    int len;
    EVP_DecryptUpdate(ctx, nullptr, &len, aad.data(), aad.size());
    
    // 设置预期标签
    EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_AEAD_SET_TAG, 16, tag.data());
    
    // 解密
    std::vector<std::uint8_t> plaintext(ciphertext.size());
    EVP_DecryptUpdate(ctx, plaintext.data(), &len, 
                      ciphertext.data(), ciphertext.size());
    
    // 验证（通过 EVP_DecryptFinal_ex）
    int ret = EVP_DecryptFinal_ex(ctx, nullptr, &len);
    
    EVP_CIPHER_CTX_free(ctx);
    
    if (ret <= 0) {
        // 认证失败
        return std::nullopt;
    }
    
    return plaintext;
}
```

### TLS 1.3 应用

在 TLS 1.3 记录层中使用 ChaCha20-Poly1305：

```cpp
// TLS 1.3 记录加密
auto encrypt_tls_record_chacha(
    const tls_context& ctx,
    tls_content_type type,
    std::span<const std::uint8_t> plaintext) -> std::vector<std::uint8_t> {
    
    // 构建记录头（作为 AAD）
    std::array<std::uint8_t, 5> aad{};
    aad[0] = static_cast<std::uint8_t>(type);
    // 版本和长度
    
    // 生成 nonce（从序列号）
    std::array<std::uint8_t, 12> nonce = compute_nonce(ctx.sequence_number);
    
    // 加密
    auto [ciphertext, tag] = chacha_encrypt(ctx.key, nonce, plaintext, aad);
    
    // 组合输出
    std::vector<std::uint8_t> record;
    record.reserve(5 + ciphertext.size() + 16);
    // 添加记录头和加密数据
    
    return record;
}
```

### 代码位置

| 源文件 | 函数/类 | 用途 |
|--------|---------|------|
| `src/prism/crypto/aead.cpp` | `aead_context` | AEAD 抽象层 |
| `src/prism/crypto/aead.cpp` | `select_optimal_algorithm` | 算法选择 |
| `include/prism/crypto/aead.hpp` | `aead_algorithm` | 算法枚举 |

### 平台检测

检测 AES-NI 支持以决定使用哪个算法：

```cpp
// x86 平台检测
bool check_aes_ni_support() {
#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__) || defined(_M_IX86)
    std::uint32_t eax, ebx, ecx, edx;
    __cpuid(1, eax, ebx, ecx, edx);
    return (ecx & bit_AES) != 0;
#else
    // 非 x86 平台，默认使用 ChaCha
    return false;
#endif
}

// ARM 平台检测
bool check_arm_crypto_support() {
#if defined(__aarch64__)
    return true;  // ARM64 通常有 AES 支持
#elif defined(__arm__)
    // 检查 ARM NEON 和 AES 扩展
    return false;  // 保守估计
#else
    return false;
#endif
}
```

### 性能对比

在不同平台上的典型性能（单线程）：

| 平台 | AES-GCM | ChaCha20-Poly1305 |
|------|---------|-------------------|
| x86 + AES-NI | 15 GB/s | 1-2 GB/s |
| x86 无 AES-NI | 0.5 GB/s | 1-2 GB/s |
| ARM Cortex-A53 | 0.8 GB/s | 1.5 GB/s |
| ARM Cortex-A72 | 1.2 GB/s | 2.5 GB/s |

## 安全考量

### 已知安全问题

**Nonce 重用**

与 AES-GCM 一样，Nonce 重用是 ChaCha20-Poly1305 最严重的安全问题：
1. 使用相同密钥和 Nonce 加密两条消息，Poly1305 密钥相同
2. 攻击者可以恢复 r 和 s
3. 攻击者可以伪造任意消息

防御措施：
- 每个密钥的生命周期内，Nonce 必须唯一
- 使用计数器作为 Nonce
- 实现自动 Nonce 管理

**短密钥**

ChaCha20-Poly1305 要求 256 位密钥。使用更短的密钥会降低安全性。

防御措施：
- 使用 HKDF 派生完整长度的密钥
- 不截断密钥

**时序攻击**

ChaCha20-Poly1305 的所有操作天然是恒定时间的，但仍需注意：
- 比较认证标签时使用恒定时间函数
- 不基于密文长度泄露信息

### 安全优势

与 AES-GCM 相比，ChaCha20-Poly1305 的安全优势：

1. **恒定时间执行**：所有操作在恒定时间内完成，无需特别小心实现
2. **无硬件依赖**：安全性不依赖于硬件实现的安全性
3. **更长的密钥**：256 位密钥提供更高的加密强度
4. **简单设计**：ARX 结构简单，易于分析和验证

### 安全建议

1. **Nonce 管理**
   - 使用计数器作为 Nonce
   - 每个密钥加密不超过 2^32 条消息
   - 实现自动 Nonce 递增

2. **密钥管理**
   - 使用完整的 256 位密钥
   - 从安全的随机源派生密钥
   - 定期轮换密钥

3. **标签验证**
   - 使用完整的 16 字节标签
   - 恒定时间比较
   - 验证失败不泄露信息

### 安全边界

| 攻击类型 | 安全强度 | 说明 |
|----------|----------|------|
| 密钥恢复 | 256 位 | 穷举密钥 |
| 标签伪造 | 128 位 | 生日攻击 |
| 流密码攻击 | >250 位 | 最佳已知攻击 |

### 实现清单

- [ ] Nonce 必须唯一
- [ ] 使用 256 位密钥
- [ ] 使用完整 16 字节标签
- [ ] 恒定时间比较标签
- [ ] 包含 AAD
- [ ] 密钥派生使用 HKDF
- [ ] 实现密钥轮换机制

## 最佳实践

### Nonce 管理

**计数器作为 Nonce**

```cpp
class chacha_nonce_manager {
public:
    chacha_nonce_manager() : counter_(0) {}
    
    std::array<std::uint8_t, 12> next_nonce() {
        std::array<std::uint8_t, 12> nonce{};
        
        // 32 位计数器，高位为 0
        std::uint32_t current = counter_.fetch_add(1);
        
        // 小端序写入
        nonce[0] = current & 0xFF;
        nonce[1] = (current >> 8) & 0xFF;
        nonce[2] = (current >> 16) & 0xFF;
        nonce[3] = (current >> 24) & 0xFF;
        // nonce[4..11] = 0
        
        // 检查溢出
        if (current == 0xFFFFFFFF) {
            throw std::runtime_error("Nonce exhausted, need key rotation");
        }
        
        return nonce;
    }
    
private:
    std::atomic<std::uint32_t> counter_;
};
```

### 密钥管理

**密钥派生**

```cpp
// 从主密钥派生 ChaCha 密钥
auto derive_chacha_key(
    std::span<const std::uint8_t> master_key,
    std::string_view label) -> std::array<std::uint8_t, 32> {
    
    // HKDF 派生 32 字节密钥
    std::vector<std::uint8_t> info;
    info.insert(info.end(), label.begin(), label.end());
    info.push_back(0x00);  // 分隔符
    
    return hkdf_expand(master_key, info, 32);
}
```

### 算法选择策略

```cpp
// 智能选择 AEAD 算法
aead_algorithm select_aead_algorithm(
    bool prefer_constant_time,
    size_t typical_payload_size) {
    
    bool has_aes_ni = check_aes_ni_support();
    
    // 恒定时间优先
    if (prefer_constant_time) {
        return aead_algorithm::chacha20_poly1305;
    }
    
    // 小数据包优先 ChaCha
    if (typical_payload_size < 256 && !has_aes_ni) {
        return aead_algorithm::chacha20_poly1305;
    }
    
    // AES-NI 平台优先 AES-GCM
    if (has_aes_ni) {
        return aead_algorithm::aes_128_gcm;
    }
    
    // 默认 ChaCha
    return aead_algorithm::chacha20_poly1305;
}
```

### 批量处理

```cpp
// 批量加密提高效率
void batch_encrypt_chacha(
    std::span<const message> messages,
    std::span<std::vector<std::uint8_t>> outputs,
    const chacha_key& key) {
    
    chacha_nonce_manager nonce_gen;
    
    for (size_t i = 0; i < messages.size(); ++i) {
        auto nonce = nonce_gen.next_nonce();
        outputs[i] = encrypt_chacha(key, nonce, messages[i].data, messages[i].aad);
    }
}
```

### 错误处理

```cpp
// 安全的解密流程
auto safe_decrypt_chacha(
    const chacha_context& ctx,
    std::span<const std::uint8_t> nonce,
    std::span<const std::uint8_t> ciphertext,
    std::span<const std::uint8_t> tag,
    std::span<const std::uint8_t> aad) -> decrypt_result {
    
    auto result = ctx.decrypt(nonce, ciphertext, tag, aad);
    
    if (!result.success) {
        // 认证失败
        // 不泄露失败原因
        return decrypt_result{.status = decrypt_status::failed};
    }
    
    return decrypt_result{
        .status = decrypt_status::success,
        .plaintext = result.plaintext
    };
}
```

### 内存管理

```cpp
// 使用 PMR 内存池提高效率
class chacha_encryptor {
public:
    chacha_encryptor(std::pmr::memory_resource* pool) 
        : pool_(pool) {}
    
    auto encrypt(
        std::span<const std::uint8_t> plaintext,
        std::span<const std::uint8_t> aad) 
        -> std::pmr::vector<std::uint8_t> {
        
        std::pmr::vector<std::uint8_t> output(pool_);
        output.reserve(plaintext.size() + 16);  // 密文 + 标签
        
        // 加密操作...
        
        return output;
    }
    
private:
    std::pmr::memory_resource* pool_;
};
```

## 常见问题

### Q1: ChaCha20-Poly1305 与 AES-GCM 如何选择？

考虑以下因素：

1. **平台硬件支持**：有 AES-NI 选择 AES-GCM，否则选择 ChaCha
2. **恒定时间要求**：需要严格恒定时间选择 ChaCha
3. **密钥长度偏好**：需要更长密钥选择 ChaCha（256 位）
4. **数据包大小**：小数据包可能 ChaCha 更优

### Q2: ChaCha20-Poly1305 的 Nonce 长度是多少？

固定为 96 位（12 字节）。这与 AES-GCM 推荐的 Nonce 长度相同。RFC 8439 明确规定使用 96 位 Nonce。

### Q3: ChaCha20-Poly1305 的密钥长度是多少？

固定为 256 位（32 字节）。密钥长度不能更改或截断。如果需要从其他密钥材料派生 ChaCha 密钥，必须使用 HKDF 或类似函数生成完整的 32 字节。

### Q4: ChaCha20-Poly1305 的认证标签可以截断吗？

RFC 8439 规定输出完整的 16 字节（128 位）标签。虽然技术上可以截断，但不推荐。截断标签会降低认证强度。

### Q5: 为什么 ChaCha20 使用 20 轮？

20 轮提供了足够的安全边界：
- 核心函数的扩散在 4-5 轨迹后基本饱和
- 最佳已知攻击约 2^250，远高于实际需求
- 20 轮是保守的选择，即使发现新的攻击也留有余地

Bernstein 也定义了 8 轮和 12 轮的变体（ChaCha8 和 ChaCha12），用于性能敏感但安全性要求稍低的场景。

### Q6: ChaCha20-Poly1305 在 ARM 平台上性能如何？

在 ARM 平台上，ChaCha20-Poly1305 通常优于 AES-GCM：
- ARM Cortex-A53：ChaCha 约 1.5 GB/s，AES-GCM 约 0.8 GB/s
- ARM Cortex-A72：ChaCha 约 2.5 GB/s，AES-GCM 约 1.2 GB/s

ARMv8 架构有 AES 扩展，但普及度不如 x86 的 AES-NI。

### Q7: ChaCha20-Poly1305 是否恒定时间？

是的。所有 ChaCha20 和 Poly1305 的操作都是恒定时间的：
- 加法、旋转、异或在恒定时间内完成
- 没有数据依赖的内存访问
- 没有数据依赖的分支

这使得 ChaCha20-Poly1305 天然防止时序攻击。

### Q8: TLS 1.3 为什么要包含 ChaCha20-Poly1305？

TLS 1.3 包含 ChaCha20-Poly1305 的原因：
1. 为不支持 AES-NI 的平台提供高性能选项
2. 提供恒定时间的 AEAD 选择
3. 提供密钥长度灵活性（256 位）
4. 增加密码套件的多样性，避免单一依赖

### Q9: Poly1305 的 r 为什么要进行位限制（clamping）？

位限制的原因：
1. 清除低 3 位：确保 r 可被 16 整除，简化实现
2. 清除高位：防止某些溢出情况
3. 清除中间位：防止某些特定的代数攻击

这些限制不影响安全性，但简化了安全分析。

### Q10: WireGuard 为什么选择 ChaCha20-Poly1305？

WireGuard 选择 ChaCha20-Poly1305 的原因：
1. 简洁：代码简单，易于审查
2. 恒定时间：天然防止时序攻击
3. 性能：在各种平台上都有良好性能
4. 安全：经过充分分析，没有已知问题
5. 无专利：完全开放，无许可限制