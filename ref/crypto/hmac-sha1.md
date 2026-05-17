---
title: "HMAC-SHA1"
category: "crypto"
type: ref
module: ref
source: "RFC 2104, RFC 3174"
tags: [密码学, hmac, sha1, 消息认证码, MAC]
created: 2026-05-15
updated: 2026-05-17
related:
  - ref/crypto/hmac-sha256
  - ref/crypto/sha256
  - ref/crypto/hkdf
  - stealth/shadowtls/auth
---

# HMAC-SHA1

**类别**: 密码学

## 概述

HMAC-SHA1（Hash-based Message Authentication Code with SHA-1）是一种基于 SHA-1 哈希函数的消息认证码算法。HMAC 由 Bellare、Canetti 和 Krawczyk 于 1996 年提出，并由 RFC 2104 标准化。它结合了加密哈希函数和密钥，用于验证消息的完整性和真实性。

HMAC 的设计目标是提供一个通用的 MAC 构造，可以使用任意安全的加密哈希函数作为底层原语。HMAC-SHA1 特指使用 SHA-1 作为底层哈希函数的实现。尽管 SHA-1 本身已经存在碰撞攻击漏洞，但 HMAC-SHA1 在消息认证的场景下仍然被认为是安全的，因为 HMAC 的安全性不依赖于哈希函数的抗碰撞性，而是依赖于其伪随机函数性质。

HMAC-SHA1 生成 160 位（20 字节）的消息认证码。这个固定长度的输出可以用于验证消息是否被篡改，以及消息是否来自声称的发送者。在 TLS 1.2 及更早版本中，HMAC-SHA1 被广泛用于记录层协议的消息认证。在 SSH 协议中，hmac-sha1 也是标准的 MAC 算法之一。

HMAC-SHA1 的工作原理是通过两次哈希运算来构造认证码。首先，将密钥与内部填充值（ipad）异或，然后与消息一起进行哈希运算。接着，将密钥与外部填充值（opad）异或，并与第一次哈希的结果一起进行第二次哈希运算。这种嵌套结构提供了额外的安全保障，使得攻击者难以通过分析输出推断密钥信息。

在现代密码学应用中，虽然 HMAC-SHA1 仍然被认为是安全的用于消息认证目的，但许多协议和系统已经开始迁移到 HMAC-SHA256 或 HMAC-SHA384，以获得更强的安全边界和更长的认证码输出。然而，在某些遗留系统或对性能有特殊要求的场景中，HMAC-SHA1 仍然是一个可行的选择。

HMAC-SHA1 的主要优势包括：算法简单高效，SHA-1 在各种平台上都有高度优化的实现；输出长度适中（20 字节），适合大多数网络协议使用；经过长期分析和广泛部署，具有成熟的安全性证明。其局限性包括：SHA-1 的碰撞攻击虽然在 HMAC 构造中不构成直接威胁，但仍可能影响某些特殊场景的安全性；160 位的输出在某些高安全需求场景可能被认为不够长。

在代理服务器的实现中，HMAC-SHA1 常用于协议级别的消息认证。例如，在 ShadowTLS 协议中，HMAC-SHA1 用于验证客户端的 SessionID 和计算数据帧的认证标签。这些应用场景利用了 HMAC-SHA1 的高效性和安全性，确保了代理通信的完整性和真实性。

HMAC-SHA1 的安全性分析需要区分两种攻击模型：伪造攻击和密钥恢复攻击。在伪造攻击中，攻击者尝试为从未见过的消息生成有效的 HMAC 值。在密钥恢复攻击中，攻击者尝试从已知的消息-HMAC 对中恢复密钥。对于 HMAC-SHA1，这两种攻击在计算上都是不可行的，最佳已知攻击的时间复杂度约为 2^80（生日攻击界限）。

值得注意的是，NIST 在 SP 800-131A 中建议在 2014 年后停止使用 SHA-1 作为数字签名算法，但 HMAC-SHA1 不受此限制影响。这是因为 HMAC 的安全性证明不依赖于哈希函数的抗碰撞性，而是依赖于其作为伪随机函数的性质。因此，HMAC-SHA1 在可预见的未来仍然可以安全使用。

在实现 HMAC-SHA1 时，需要注意避免侧信道攻击。特别是，密钥处理和哈希计算应该在恒定时间内完成，以防止通过时间分析推断密钥信息。现代加密库通常已经实现了这些保护措施，但在自行实现时需要特别小心。

HMAC-SHA1 的另一个重要应用是在密钥派生函数中。HKDF（HMAC-based Extract-and-Expand Key Derivation Function）可以使用 HMAC-SHA1 作为底层原语，从输入密钥材料中派生出任意长度的密钥。虽然 HKDF-SHA256 更为常用，但在某些需要与旧系统兼容的场景中，HKDF-SHA1 仍然有其用武之地。

总的来说，HMAC-SHA1 是一个经过充分研究和广泛部署的消息认证算法。尽管 SHA-1 哈希函数本身已经不再推荐用于数字签名等需要抗碰撞性的场景，但 HMAC-SHA1 在消息认证领域仍然是安全的。在新系统中，建议优先考虑 HMAC-SHA256 或 HMAC-SHA384，但在需要与旧系统兼容或对性能有特殊要求的场景中，HMAC-SHA1 仍然是一个合理的选择。

## 算法原理

### HMAC 结构

HMAC（Hash-based Message Authentication Code）是一种基于加密哈希函数的消息认证码构造。HMAC 的核心思想是利用哈希函数的压缩特性，将密钥和消息混合在一起，生成固定长度的认证标签。

HMAC 的通用公式为：

```
HMAC(K, M) = H((K ⊕ opad) || H((K ⊕ ipad) || M))
```

其中：
- H 是加密哈希函数（在 HMAC-SHA1 中为 SHA-1）
- K 是密钥
- M 是消息
- ipad 是内部填充值（0x36 重复块大小次）
- opad 是外部填充值（0x5C 重复块大小次）
- || 表示连接操作
- ⊕ 表示异或操作

对于 HMAC-SHA1，具体的参数如下：
- 块大小（B）= 64 字节（512 位）
- 输出大小（L）= 20 字节（160 位）
- ipad = 0x36 重复 64 次 = 36363636...36（64 字节）
- opad = 0x5C 重复 64 次 = 5C5C5C5C...5C（64 字节）

### SHA-1 哈希函数

SHA-1（Secure Hash Algorithm 1）是由美国国家标准与技术研究院（NIST）设计的加密哈希函数，于 1995 年发布作为联邦信息处理标准（FIPS 180-1）。SHA-1 生成 160 位（20 字节）的哈希值。

SHA-1 的处理流程如下：

**第一步：消息填充**

将原始消息填充到长度为 448 mod 512 位（即填充后的消息长度比 512 的倍数少 64 位）。填充规则如下：
1. 在消息末尾添加一个 1 位
2. 添加若干个 0 位，直到消息长度为 448 mod 512
3. 在末尾附加 64 位（8 字节）的原始消息长度（大端序）

例如，对于消息 "abc"：
- 原始消息长度：24 位（3 字节 × 8 位）
- 填充后：24 位 + 1 位 + 423 位 + 64 位 = 512 位

**第二步：初始化哈希值**

SHA-1 使用 5 个 32 位寄存器来保存中间哈希状态，初始值为特定的常量（由平方根分数部分导出）：

```
H0 = 0x67452301
H1 = 0xEFCDAB89
H2 = 0x98BADCFE
H3 = 0x10325476
H4 = 0xC3D2E1F0
```

**第三步：消息调度（消息扩展）**

对于每个 512 位（64 字节）的消息块，将其扩展为 80 个 32 位的字 W0, W1, ..., W79：

```
Wt = Mt                    (0 ≤ t ≤ 15)
Wt = (Wt-3 ⊕ Wt-8 ⊕ Wt-14 ⊕ Wt-16) <<< 1  (16 ≤ t ≤ 79)
```

其中 <<< 表示循环左移，⊕ 表示异或操作。

**第四步：压缩函数**

SHA-1 使用 80 轮迭代处理每个消息块。每轮更新 5 个工作变量 a, b, c, d, e：

对于轮次 t = 0 到 79：
- 根据轮次选择函数 f 和常量 K
- 计算临时值：T = (a <<< 5) + f + e + K + Wt
- 更新工作变量：e = d, d = c, c = b <<< 30, b = a, a = T

轮函数和常量根据轮次范围不同：

| 轮次范围 | 函数 f(b, c, d) | 常量 K |
|----------|----------------|--------|
| 0-19 | (b ∧ c) ⊕ (¬b ∧ d) | 0x5A827999 |
| 20-39 | b ⊕ c ⊕ d | 0x6ED9EBA1 |
| 40-59 | (b ∧ c) ⊕ (b ∧ d) ⊕ (c ∧ d) | 0x8F1BBCDC |
| 60-79 | b ⊕ c ⊕ d | 0xCA62C1D6 |

其中 ∧ 表示与，⊕ 表示异或，¬ 表示按位取反。

**第五步：更新哈希值**

处理完每个消息块后，将工作变量加到哈希值上：

```
H0 = H0 + a
H1 = H1 + b
H2 = H2 + c
H3 = H3 + d
H4 = H4 + e
```

**第六步：输出**

最终哈希值为 H0 || H1 || H2 || H3 || H4（共 160 位，20 字节）。

### HMAC-SHA1 完整算法

将上述 HMAC 结构与 SHA-1 结合，得到 HMAC-SHA1 的完整算法：

**输入**：密钥 K（任意长度），消息 M（任意长度）

**输出**：160 位（20 字节）的消息认证码

**步骤**：

1. **密钥预处理**：
   - 如果 |K| > 64 字节：K' = SHA1(K)
   - 如果 |K| < 64 字节：K' = K || 00...00（用零填充到 64 字节）
   - 如果 |K| = 64 字节：K' = K

2. **计算内部哈希**：
   - 计算内部填充密钥：Ki = K' ⊕ ipad = K' ⊕ (0x36 || 0x36 || ... || 0x36)
   - 计算内部哈希：Hi = SHA1(Ki || M)

3. **计算外部哈希**：
   - 计算外部填充密钥：Ko = K' ⊕ opad = K' ⊕ (0x5C || 0x5C || ... || 0x5C)
   - 计算最终 HMAC：HMAC-SHA1(K, M) = SHA1(Ko || Hi)

### 安全性分析

**安全性证明**

HMAC 的安全性基于以下假设：
1. 底层哈希函数的压缩函数是伪随机函数
2. 哈希函数具有良好的扩散特性

Bellare、Canetti 和 Krawczyk 在 1996 年证明了 HMAC 的安全性可以归约到压缩函数的伪随机性。这意味着如果压缩函数是安全的 PRF，那么 HMAC 也是安全的。

**为什么 SHA-1 碰撞攻击不影响 HMAC-SHA1**

SHA-1 的碰撞攻击（如 SHAttered，2017 年）利用了哈希函数的消息扩展特性和特定的差分路径。然而，HMAC 的安全性不依赖于哈希函数的抗碰撞性，而是依赖于其作为 PRF 的性质。

具体来说：
1. HMAC 使用密钥作为输入的一部分，攻击者无法控制密钥
2. HMAC 的嵌套结构增加了攻击复杂度
3. 碰撞攻击需要找到两个不同消息产生相同哈希值，但 HMAC 需要找到能产生特定 MAC 值的伪造消息

因此，即使 SHA-1 存在碰撞漏洞，HMAC-SHA1 仍然被认为是安全的消息认证码。

**安全边界**

- 伪造攻击复杂度：约 2^80（生日攻击）
- 密钥恢复攻击复杂度：约 2^160（穷举密钥）
- 输出长度：160 位

**推荐实践**

虽然 HMAC-SHA1 仍然安全，但建议：
1. 新系统优先使用 HMAC-SHA256
2. 在需要更高安全性的场景使用 HMAC-SHA384 或 HMAC-SHA512
3. 定期轮换密钥
4. 避免在密钥材料不足的情况下直接使用 HMAC

## 在 Prism 中的应用

Prism 在多个模块中使用 HMAC-SHA1，主要集中在 ShadowTLS 协议实现中。

### ShadowTLS 认证模块

在 ShadowTLS 协议中，HMAC-SHA1 用于多个关键安全操作：

**SessionID 验证**

ShadowTLS 协议使用 HMAC-SHA1 来验证客户端提供的 SessionID 是否有效。服务端维护一个预计算的 HMAC 表，客户端需要提供正确的 SessionID 才能通过认证：

```cpp
// 伪代码示例
bool verify_client_hello(
    std::span<const std::uint8_t> session_id,
    std::span<const std::uint8_t> expected_hmac) {
    // 使用 HMAC-SHA1 计算会话ID的认证标签
    auto computed = compute_hmac_sha1(secret_key_, session_id);
    // 常量时间比较以防止时序攻击
    return constant_time_compare(computed, expected_hmac);
}
```

**HMAC 标签计算**

ShadowTLS 协议中的数据帧使用 HMAC-SHA1 作为认证标签。每个数据帧都包含一个 HMAC 值，用于验证数据完整性和来源真实性：

```cpp
// 计算 HMAC 标签
auto compute_frame_hmac(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> payload) {
    return hmac_sha1(key, payload);
}
```

**写入 HMAC 计算**

在向客户端发送数据时，ShadowTLS 使用 HMAC-SHA1 计算写入标签：

```cpp
// 计算写入 HMAC
std::vector<std::uint8_t> compute_write_hmac(
    std::span<const std::uint8_t> write_key,
    std::span<const std::uint8_t> data) {
    return hmac_sha1(write_key, data);
}
```

### 代码位置

| 源文件 | 函数/类 | 用途 |
|--------|---------|------|
| `src/prism/stealth/shadowtls/auth.cpp` | `verify_client_hello` | 验证客户端 SessionID |
| `src/prism/stealth/shadowtls/auth.cpp` | `compute_hmac` | 计算 HMAC 标签 |
| `src/prism/stealth/shadowtls/auth.cpp` | `verify_frame_hmac` | 验证数据帧 HMAC |
| `src/prism/stealth/shadowtls/auth.cpp` | `compute_write_hmac` | 计算写入 HMAC |

### 依赖关系

HMAC-SHA1 在 Prism 中的使用依赖于底层加密库（BoringSSL）提供的 SHA-1 实现。Prism 的加密抽象层确保了：

1. 恒定时间操作以防止时序攻击
2. 安全的内存处理（使用后清零密钥材料）
3. 正确的错误处理和传播

### 与其他算法的配合

在某些场景中，HMAC-SHA1 与其他加密原语配合使用：

- 与 HKDF 配合进行密钥派生（虽然更常用 HMAC-SHA256）
- 与 AES-GCM 或 ChaCha20-Poly1305 配合提供额外的认证层
- 与 X25519 ECDHE 配合进行密钥确认

## 安全考量

### 已知安全问题

**SHA-1 碰撞攻击**

2017 年，Google 和 CWI Amsterdam 公布了 SHAttered 攻击，成功找到了两个 SHA-1 哈希值相同的 PDF 文件。然而，这个攻击针对的是碰撞抵抗性，不直接影响 HMAC-SHA1 的安全性。

HMAC 的安全性基于 PRF（伪随机函数）性质，而非抗碰撞性。因此，HMAC-SHA1 仍然被认为是安全的消息认证码。

**长度扩展攻击**

SHA-1 本身易受长度扩展攻击，但 HMAC 的构造方式天然地防止了这种攻击。因为攻击者不知道内部哈希的完整输出（只看到经过外部哈希处理后的结果），无法进行长度扩展。

**时序攻击**

在实现 HMAC-SHA1 比较操作时，必须使用恒定时间比较函数。普通的字符串比较会在第一个不匹配字节时提前返回，这会泄露信息给攻击者。Prism 使用恒定时间比较来防止此类攻击。

### 安全建议

1. **密钥管理**
   - 使用足够长度的密钥（推荐至少 16 字节）
   - 定期轮换密钥
   - 不同用途使用不同密钥

2. **实现安全**
   - 使用经过审查的加密库实现
   - 使用恒定时间比较函数
   - 使用后安全清零密钥材料

3. **协议设计**
   - 在新协议中优先使用 HMAC-SHA256
   - 为 HMAC-SHA1 实现制定明确的弃用计划
   - 考虑未来迁移到更长的认证标签

### 安全边界

| 攻击类型 | 复杂度 | 说明 |
|----------|--------|------|
| 伪造攻击 | 2^80 | 生日攻击界限 |
| 密钥恢复 | 2^L | L 为密钥长度 |
| 长度扩展 | 不可行 | HMAC 天然防御 |

### 迁移建议

虽然 HMAC-SHA1 仍然安全，但建议新项目使用 HMAC-SHA256。迁移路径：

1. 同时支持 HMAC-SHA1 和 HMAC-SHA256
2. 新连接优先协商 HMAC-SHA256
3. 逐步淘汰 HMAC-SHA1 支持

## 最佳实践

### 密钥管理

1. **密钥生成**
   ```cpp
   // 使用密码学安全的随机数生成器
   std::array<std::uint8_t, 32> key;
   RAND_bytes(key.data(), key.size());
   ```

2. **密钥存储**
   - 密钥不应以明文形式存储在配置文件中
   - 考虑使用密钥管理系统或硬件安全模块（HSM）
   - 使用后立即清零密钥材料

3. **密钥轮换**
   - 定期轮换密钥（建议每 90 天）
   - 密钥泄露后立即轮换
   - 支持多密钥同时生效以实现无缝轮换

### 实现注意事项

1. **恒定时间比较**
   ```cpp
   // 错误：不安全
   if (hmac == expected) { ... }
   
   // 正确：恒定时间比较
   if (CRYPTO_memcmp(hmac.data(), expected.data(), hmac.size()) == 0) { ... }
   ```

2. **内存清理**
   ```cpp
   // 使用后清理密钥材料
   OPENSSL_cleanse(key.data(), key.size());
   ```

3. **错误处理**
   - 认证失败时不应泄露具体错误原因
   - 记录足够的日志用于审计
   - 防止通过错误消息进行信息泄露

### 性能优化

1. **批量处理**
   - 对于多个消息，考虑批量计算 HMAC
   - 避免频繁初始化和清理

2. **SIMD 加速**
   - 现代 CPU 支持 SHA 扩展
   - 使用硬件加速的 SHA-1 实现

3. **缓存友好**
   - 保持工作集在 L1 缓存内
   - 避免频繁的内存分配

### 代码示例

**基本 HMAC-SHA1 计算**
```cpp
#include <openssl/hmac.h>
#include <vector>
#include <span>

std::vector<std::uint8_t> hmac_sha1(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> message) {
    std::vector<std::uint8_t> result(SHA_DIGEST_LENGTH);
    
    HMAC_CTX* ctx = HMAC_CTX_new();
    HMAC_Init_ex(ctx, key.data(), key.size(), EVP_sha1(), nullptr);
    HMAC_Update(ctx, message.data(), message.size());
    HMAC_Final(ctx, result.data(), nullptr);
    HMAC_CTX_free(ctx);
    
    return result;
}
```

**恒定时间验证**
```cpp
#include <openssl/crypto.h>

bool verify_hmac_sha1(
    std::span<const std::uint8_t> key,
    std::span<const std::uint8_t> message,
    std::span<const std::uint8_t> expected) {
    
    auto computed = hmac_sha1(key, message);
    
    // 必须使用恒定时间比较
    if (computed.size() != expected.size()) {
        return false;
    }
    
    return CRYPTO_memcmp(computed.data(), expected.data(), computed.size()) == 0;
}
```

## 常见问题

### Q1: HMAC-SHA1 是否仍然安全？

是的，HMAC-SHA1 在消息认证场景下仍然被认为是安全的。虽然 SHA-1 本身存在碰撞攻击，但 HMAC 的安全性不依赖于哈希函数的抗碰撞性。然而，对于新项目，建议使用 HMAC-SHA256 以获得更大的安全边界。

### Q2: HMAC-SHA1 的输出长度是多少？

HMAC-SHA1 的输出长度为 20 字节（160 位）。这是由 SHA-1 的输出长度决定的。如果需要更短的认证标签，可以安全地截断输出，但每个截断位会降低安全强度。

### Q3: 密钥长度有什么要求？

HMAC-SHA1 可以接受任意长度的密钥。如果密钥超过 64 字节，会被先进行 SHA-1 哈希。如果密钥短于 64 字节，会被用零填充。推荐使用至少 16 字节的密钥以获得足够的安全强度。

### Q4: 为什么 HMAC-SHA1 不受 SHA-1 碰撞攻击影响？

HMAC 的安全性基于哈希函数作为伪随机函数的性质，而非其抗碰撞性。碰撞攻击需要找到两个不同的消息产生相同的哈希值，但这需要在不知道密钥的情况下完成。由于密钥参与了两次哈希运算（内部和外部），攻击者无法控制内部哈希的输入，因此碰撞攻击不适用。

### Q5: HMAC-SHA1 与 HMAC-SHA256 有什么区别？

主要区别在于底层哈希函数：
- HMAC-SHA1 使用 SHA-1，输出 160 位
- HMAC-SHA256 使用 SHA-256，输出 256 位
- HMAC-SHA256 提供更大的安全边界
- HMAC-SHA256 的计算成本略高（更多轮次和更大的状态）

### Q6: 可以截断 HMAC-SHA1 输出吗？

可以。RFC 2104 明确允许截断 HMAC 输出。截断后的安全强度为 min(截断后长度 × 8, 80) 位。例如，截断到 12 字节提供约 96 位安全强度，但受限于生日攻击的 80 位上限。

### Q7: HMAC-SHA1 适用于密钥派生吗？

HMAC-SHA1 可以用于密钥派生，例如在 HKDF 中。然而，对于新系统，建议使用 HMAC-SHA256 或 HMAC-SHA384 作为 HKDF 的底层哈希函数，以获得更长的派生密钥和更大的安全边界。

### Q8: 如何在 TLS 中配置 HMAC-SHA1？

在 TLS 1.2 中，HMAC-SHA1 作为 HMAC-SHA1 密码套件的一部分使用。然而，现代 TLS 配置通常禁用 SHA-1 相关的密码套件。TLS 1.3 完全移除了对 SHA-1 的支持，所有密码套件都使用 SHA-256 或 SHA-384。

### Q9: HMAC-SHA1 能防止重放攻击吗？

HMAC-SHA1 本身不提供重放保护。认证码只能验证消息的完整性和来源，但不能检测消息是否被重复发送。要防止重放攻击，需要额外的机制，如序列号、时间戳或随机数（nonce）。

### Q10: ShadowTLS 为什么使用 HMAC-SHA1？

ShadowTLS 协议设计中使用 HMAC-SHA1 主要是因为：
1. 兼容性：与目标 TLS 服务器的行为保持一致
2. 性能：SHA-1 在各种平台上都有高度优化的实现
3. 安全性：在 HMAC 构造中，SHA-1 仍然足够安全
4. 伪装：使用与真实 TLS 相同的算法增强隐蔽性

## 参考资料

- [RFC 2104 - HMAC: Keyed-Hashing for Message Authentication](https://tools.ietf.org/html/rfc2104)
- [RFC 3174 - US Secure Hash Algorithm 1 (SHA1)](https://tools.ietf.org/html/rfc3174)
- [NIST SP 800-107 - Recommendation for Applications Using Approved Hash Algorithms](https://csrc.nist.gov/publications/detail/sp/800-107/final)
- [Bellare, Canetti, Krawczyk - Keying Hash Functions for Message Authentication](https://cseweb.ucsd.edu/~mihir/papers/hmac.html)

## 发展历史

HMAC 的设计始于 1995 年，由 Bellare、Canetti 和 Krawczyk 三位密码学家完成。他们的目标是创建一个通用的 MAC 构造，可以使用任意安全的哈希函数作为底层原语。这一设计的灵活性使得 HMAC 可以轻松地从 SHA-1 迁移到 SHA-256 或其他哈希函数。

1997 年，HMAC 被 RFC 2104 正式标准化。此后，HMAC-SHA1 成为 TLS 1.0、TLS 1.1 和 TLS 1.2 中广泛使用的 MAC 算法。SSH 协议也采用 HMAC-SHA1 作为默认的 MAC 算法。

2005 年，密码学家开始研究 SHA-1 的安全性。虽然理论上的碰撞攻击被发现，但这些攻击不影响 HMAC-SHA1 的安全性。2017 年的 SHAttered 攻击证实了 SHA-1 的碰撞漏洞，但密码学社区一致认为 HMAC-SHA1 在消息认证场景下仍然安全。

2014 年，NIST 在 SP 800-131A 中建议停止使用 SHA-1 作为数字签名算法。然而，HMAC-SHA1 不受此限制影响，因为 HMAC 的安全性与数字签名的安全性依赖不同的属性。

2020 年代，虽然 HMAC-SHA256 成为新系统的默认选择，但 HMAC-SHA1 在遗留系统和某些特殊场景中仍然有应用。ShadowTLS 等现代协议选择 HMAC-SHA1 的原因是其与目标 TLS 服务器的兼容性和高性能实现。

## 与其他 MAC 算法的比较

| 算法 | 输出长度 | 安全强度 | 性能 | 适用场景 |
|------|----------|----------|------|----------|
| HMAC-SHA1 | 160 位 | 80 位 | 高 | 遗留系统 |
| HMAC-SHA256 | 256 位 | 128 位 | 中 | 新系统 |
| HMAC-SHA384 | 384 位 | 192 位 | 中低 | 高安全需求 |
| Poly1305 | 128 位 | 128 位 | 高 | AEAD 配合 |
| CMAC-AES | 128 位 | 128 位 | 中 | 硬件加速场景 |

## 相关知识

- [[ref/crypto/hmac-sha256|HMAC-SHA256]] — 更安全的 HMAC 变体
- [[ref/crypto/sha256|SHA-256]] — 更安全的哈希函数
- [[ref/crypto/hkdf|HKDF]] — 使用 HMAC 的密钥派生函数
- [[ref/protocol/tls-sessionid|TLS SessionID]] — ShadowTLS 如何使用 SessionID