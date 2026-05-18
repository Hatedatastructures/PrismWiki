---
title: "HKDF"
category: "crypto"
type: ref
module: ref
source: "RFC 5869"
tags: [密码学, hkdf, 密钥派生]
created: 2026-05-15
updated: 2026-05-17
layer: ref
---

# HKDF

**类别**: 密码学

## 概述

HKDF（HMAC-based Extract-and-Expand Key Derivation Function）是一种基于 HMAC 的密钥派生函数，由 H. Krawczyk 和 P. Eronen 在 RFC 5869 中标准化。HKDF 的设计目标是从输入密钥材料（Input Keying Material, IKM）中安全地派生出用于各种加密用途的密钥。

HKDF 的核心思想是将密钥派生过程分为两个独立的阶段：提取（Extract）和扩展（Expand）。这种两阶段设计使得 HKDF 能够处理各种类型的输入密钥材料，包括熵分布不均匀的密钥材料，并生成具有所需长度和属性的输出密钥。

提取阶段从输入密钥材料中提取固定长度的伪随机密钥（Pseudorandom Key, PRK）。这一阶段使用 HMAC 将输入密钥材料与可选的盐值混合，生成一个均匀分布的伪随机值。提取阶段的作用是"浓缩"输入密钥材料中的熵，即使输入熵分布不均匀，也能产生高质量的伪随机输出。

扩展阶段从伪随机密钥中派生出所需长度和数量的输出密钥。这一阶段使用 HMAC 的迭代调用，结合可选的上下文信息，生成任意长度的密钥材料。扩展阶段支持从单个 PRK 派生出多个用途不同的密钥，每个密钥可以通过不同的 info 参数来区分。

HKDF 的设计具有多种安全优势。首先，它基于 HMAC 的安全性，继承了 HMAC 的所有安全保证。其次，提取阶段可以处理弱密钥材料，将其转换为强密钥。第三，扩展阶段保证派生出的不同密钥在密码学上是独立的。第四，盐值的使用增加了密钥派生的灵活性和安全性。

在 TLS 1.3 中，HKDF 是核心的密钥派生机制。TLS 1.3 使用 HKDF-Extract 和 HKDF-Expand-Label 函数来派生所有握手和应用数据密钥。这种标准化的密钥派生方法简化了协议设计，提高了安全性，并减少了实现错误的可能性。

HKDF 支持多种哈希函数作为底层 HMAC 的原语。常见选择包括 SHA-256、SHA-384 和 SHA-512。哈希函数的选择决定了 PRK 的长度和 HMAC 的安全强度。对于大多数应用，HKDF-SHA256 是最常用的选择，提供 256 位的安全强度。

HKDF 的盐值（Salt）是一个可选参数，但强烈推荐使用。盐值的作用是增加密钥派生的随机性和安全性。使用不同的盐值可以从相同的 IKM 派生出不同的 PRK，这对于密钥分离和密钥轮换非常有用。如果没有盐值，可以使用全零的默认值，但这种情况下的安全性略低于使用盐值。

HKDF 的信息参数（Info）用于在扩展阶段区分不同的密钥用途。例如，在 TLS 1.3 中，info 参数包含密钥用途标签、握手上下文哈希等信息。这种设计使得从同一个 PRK 派生出的不同密钥具有明确的用途标识，避免了密钥误用。

在代理服务器和网络协议实现中，HKDF 广泛用于密钥派生。Prism 在 Reality 协议的密钥派生、TLS 会话密钥生成等多个场景中使用 HKDF。HKDF 提供了标准化的密钥派生方法，确保派生出的密钥具有正确的长度和独立的安全属性。

HKDF 还支持从单个 IKM 派生出多个密钥。这在需要多个加密密钥的场景中非常有用，例如 TLS 1.3 需要派生出客户端和服务端的加密密钥、MAC 密钥等。通过使用不同的 info 参数，可以确保这些密钥在密码学上是独立的。

HKDF 的实现相对简单，只需要调用 HMAC 函数即可。然而，需要注意一些安全细节：盐值和 info 参数的正确使用、输出长度的限制、PRK 和输出密钥的安全存储等。现代加密库通常提供 HKDF 的完整实现，开发者应优先使用这些经过验证的实现。

总的来说，HKDF 是一种设计精良的密钥派生函数，在安全性、灵活性和实现简单性之间取得了良好的平衡。理解 HKDF 的工作原理和正确使用方法对于安全地实现各种加密协议至关重要。

## 算法原理

### HKDF 结构

HKDF 由两个阶段组成：

1. **提取阶段（HKDF-Extract）**：从输入密钥材料中提取伪随机密钥
2. **扩展阶段（HKDF-Expand）**：从伪随机密钥中派生输出密钥材料

这种两阶段设计有以下优势：
- 处理各种类型的输入密钥材料（强或弱）
- 支持从单个 IKM 派生多个独立密钥
- 提供清晰的密码学分离

### 提取阶段

提取阶段使用 HMAC 从输入密钥材料中生成伪随机密钥：

```
PRK = HMAC-Hash(salt, IKM)
```

其中：
- PRK：伪随机密钥（输出）
- salt：盐值（可选，推荐使用）
- IKM：输入密钥材料
- Hash：底层哈希函数（如 SHA-256）

**盐值的作用**：

盐值在提取阶段有以下重要作用：
1. **增加随机性**：不同的盐值产生不同的 PRK
2. **密钥分离**：同一 IKM 可以派生出多个不同的 PRK
3. **抵抗弱 IKM**：即使 IKM 熵不足，盐值也能增加安全性
4. **协议绑定**：盐值可以将密钥派生绑定到特定协议实例

如果没有提供盐值，使用长度等于 Hash 输出长度的全零字符串作为默认盐值。但推荐总是使用盐值，特别是在 IKM 可能较弱的情况下。

**PRK 的长度**：

PRK 的长度等于底层哈希函数的输出长度：
- HKDF-SHA256：PRK 为 32 字节
- HKDF-SHA384：PRK 为 48 字节
- HKDF-SHA512：PRK 为 64 字节

### 扩展阶段

扩展阶段从 PRK 中生成所需的输出密钥材料：

```
T(0) = empty string (zero length)
T(i) = HMAC-Hash(PRK, T(i-1) | info | i)   [for i = 1..N]

OKM = T(1) | T(2) | ... | T(N)

其中：
N = ceil(L / HashLen)
T(N) 的前 L mod HashLen 字节用于 OKM 的最后部分
```

其中：
- OKM：输出密钥材料（输出）
- PRK：伪随机密钥（来自提取阶段）
- info：上下文和应用特定信息（可选）
- L：所需的 OKM 长度（字节）
- i：计数器（单字节，从 1 开始）
- HashLen：哈希函数输出长度

**Info 参数的作用**：

Info 参数用于：
1. **密钥用途标识**：区分派生出的不同密钥（如加密密钥、MAC 密钥）
2. **协议绑定**：将密钥绑定到特定协议实例
3. **应用特定信息**：包含应用程序特定的上下文信息

Info 参数通常是协议定义的结构化数据，例如：
- 密钥用途标签（如 "key", "iv"）
- 协议标识符
- 会话标识符
- 消息序列号

**输出长度限制**：

由于计数器 i 是单字节（最大值 255），HKDF-Expand 的最大输出长度为：
```
L_max = 255 * HashLen
```

对于 HKDF-SHA256，L_max = 255 * 32 = 8160 字节。

对于超过这个长度的需求，需要多次调用 HKDF 或使用不同的 info 参数。

### TLS 1.3 HKDF-Expand-Label

TLS 1.3 定义了 HKDF-Expand-Label 函数，用于标准化的密钥派生：

```
HKDF-Expand-Label(Secret, Label, Context, Length) =
    HKDF-Expand(Secret, HkdfLabel, Length)

其中 HkdfLabel 结构为：
struct {
    uint16 length = Length;
    opaque label<7..255> = "tls13 " + Label;
    opaque context<0..255> = Context;
} HkdfLabel;
```

Label 的前缀 "tls13 " 用于将 TLS 1.3 密钥与其他用途的密钥区分开来。

TLS 1.3 使用多种标签来派生不同用途的密钥：

| 标签 | 用途 |
|------|------|
| "key" | 加密密钥 |
| "iv" | 初始向量/Nonce |
| "res master" | 主恢复密钥 |
| "finished" | Finished 消息验证密钥 |
| "c hs traffic" | 客户端握手密钥 |
| "s hs traffic" | 服务端握手密钥 |
| "c ap traffic" | 客户端应用密钥 |
| "s ap traffic" | 服务端应用密钥 |

### 数学分析

**安全性基础**

HKDF 的安全性基于 HMAC 的安全性：

1. **PRF 安全性**：HMAC 是安全的伪随机函数（PRF）
2. **随机性提取**：提取阶段通过 HMAC 的伪随机性将 IKM 转换为均匀分布的 PRK
3. **密钥独立性**：扩展阶段保证派生出的不同密钥在密码学上独立

**熵分析**

假设 IKM 包含 n 位的熵，salt 包含 s 位的熵（如果使用随机盐）：
- PRK 的熵约为 min(n, HashLen * 8) 位
- OKM 的熵约为 min(n + s, L * 8) 位

如果 IKM 的熵分布不均匀，提取阶段通过 HMAC 的混合操作将其均匀化。

**密钥分离**

使用不同的 info 参数派生的密钥在密码学上是独立的：
- 攻击者无法从一个密钥推断另一个密钥
- 密钥之间的相关性在密码学上可以忽略

### 完整算法示例

**HKDF-SHA256 完整实现**

```cpp
// 提取阶段
std::vector<std::uint8_t> hkdf_extract_sha256(
    std::span<const std::uint8_t> salt,
    std::span<const std::uint8_t> ikm) {
    
    // 如果 salt 为空，使用全零
    std::vector<std::uint8_t> default_salt(32, 0);
    std::span<const std::uint8_t> actual_salt = salt.empty() 
        ? default_salt : salt;
    
    // PRK = HMAC-SHA256(salt, IKM)
    return hmac_sha256(actual_salt, ikm);
}

// 扩展阶段
std::vector<std::uint8_t> hkdf_expand_sha256(
    std::span<const std::uint8_t> prk,
    std::span<const std::uint8_t> info,
    size_t length) {
    
    const size_t hash_len = 32;
    
    // 检查长度限制
    if (length > 255 * hash_len) {
        throw std::runtime_error("HKDF output length exceeds limit");
    }
    
    // 计算迭代次数
    size_t n = (length + hash_len - 1) / hash_len;
    
    std::vector<std::uint8_t> okm;
    std::vector<std::uint8_t> t_prev;
    
    for (size_t i = 1; i <= n; ++i) {
        // T(i) = HMAC-SHA256(PRK, T(i-1) | info | i)
        std::vector<std::uint8_t> input;
        input.insert(input.end(), t_prev.begin(), t_prev.end());
        input.insert(input.end(), info.begin(), info.end());
        input.push_back(static_cast<std::uint8_t>(i));
        
        t_prev = hmac_sha256(prk, input);
        
        // 添加到输出
        size_t bytes_to_add = (i == n) 
            ? (length - okm.size()) : hash_len;
        okm.insert(okm.end(), t_prev.begin(), t_prev.begin() + bytes_to_add);
    }
    
    return okm;
}

// 完整 HKDF
std::vector<std::uint8_t> hkdf_sha256(
    std::span<const std::uint8_t> salt,
    std::span<const std::uint8_t> ikm,
    std::span<const std::uint8_t> info,
    size_t length) {
    
    auto prk = hkdf_extract_sha256(salt, ikm);
    return hkdf_expand_sha256(prk, info, length);
}
```

### 安全性分析

**安全强度**

HKDF 的安全强度取决于底层哈希函数：
- HKDF-SHA256：256 位安全强度
- HKDF-SHA384：384 位安全强度
- HKDF-SHA512：512 位安全强度

**安全性证明**

HKDF 的安全性由 Krawczyk 在 2010 年的论文中证明：
1. 提取阶段满足随机性提取器的安全定义
2. 扩展阶段满足伪随机函数扩展的安全定义
3. 两阶段组合满足整体密钥派生的安全目标

**已知限制**

1. **输出长度限制**：最大 255 * HashLen 字节
2. **Info 长度限制**：info 参数应保持在合理范围内（通常小于 HashLen）
3. **弱 IKM 处理**：如果 IKM 熵极低（如少于安全强度的一半），安全性降低

## 在 Prism 中的应用

Prism 在多个模块中使用 HKDF，主要用于 Reality 协议和 TLS 密钥派生。

### Reality 密钥派生

Reality 协议使用 HKDF 派生所有会话密钥：

**握手密钥派生**

```cpp
// Reality 握手密钥派生
struct handshake_keys {
    std::array<std::uint8_t, 16> client_key;
    std::array<std::uint8_t, 12> client_iv;
    std::array<std::uint8_t, 16> server_key;
    std::array<std::uint8_t, 12> server_iv;
};

auto derive_handshake_keys(
    std::span<const std::uint8_t> shared_secret,  // ECDHE 共享密钥
    std::span<const std::uint8_t> handshake_hash) -> handshake_keys {
    
    // 使用 HKDF-Extract 提取伪随机密钥
    auto early_secret = hkdf_extract({}, {});
    auto handshake_secret = hkdf_extract(early_secret, shared_secret);
    
    // 使用 HKDF-Expand-Label 派生各个密钥
    handshake_keys keys;
    
    // 客户端加密密钥
    auto c_hs_secret = hkdf_expand_label(handshake_secret, 
                                          "c hs traffic", handshake_hash, 32);
    keys.client_key = hkdf_expand_label(c_hs_secret, "key", {}, 16);
    keys.client_iv = hkdf_expand_label(c_hs_secret, "iv", {}, 12);
    
    // 服务端加密密钥
    auto s_hs_secret = hkdf_expand_label(handshake_secret, 
                                          "s hs traffic", handshake_hash, 32);
    keys.server_key = hkdf_expand_label(s_hs_secret, "key", {}, 16);
    keys.server_iv = hkdf_expand_label(s_hs_secret, "iv", {}, 12);
    
    return keys;
}
```

**应用密钥派生**

```cpp
// Reality 应用密钥派生
struct application_keys {
    std::array<std::uint8_t, 16> client_key;
    std::array<std::uint8_t, 12> client_iv;
    std::array<std::uint8_t, 16> server_key;
    std::array<std::uint8_t, 12> server_iv;
};

auto derive_application_keys(
    std::span<const std::uint8_t> master_secret,
    std::span<const std::uint8_t> handshake_hash) -> application_keys {
    
    // 派生主密钥
    auto derived_secret = hkdf_expand_label(master_secret, 
                                             "derived", {}, 32);
    
    // 派生应用密钥
    application_keys keys;
    
    auto c_ap_secret = hkdf_expand_label(derived_secret, 
                                          "c ap traffic", handshake_hash, 32);
    keys.client_key = hkdf_expand_label(c_ap_secret, "key", {}, 16);
    keys.client_iv = hkdf_expand_label(c_ap_secret, "iv", {}, 12);
    
    auto s_ap_secret = hkdf_expand_label(derived_secret, 
                                          "s ap traffic", handshake_hash, 32);
    keys.server_key = hkdf_expand_label(s_ap_secret, "key", {}, 16);
    keys.server_iv = hkdf_expand_label(s_ap_secret, "iv", {}, 12);
    
    return keys;
}
```

### HKDF-Expand-Label 实现

```cpp
// TLS 1.3 HKDF-Expand-Label
std::vector<std::uint8_t> hkdf_expand_label(
    std::span<const std::uint8_t> secret,
    std::string_view label,
    std::span<const std::uint8_t> context,
    size_t length) {
    
    // 构建 HkdfLabel 结构
    std::vector<std::uint8_t> hkdf_label;
    
    // Length (2 字节, 大端序)
    hkdf_label.push_back(static_cast<std::uint8_t>(length >> 8));
    hkdf_label.push_back(static_cast<std::uint8_t>(length));
    
    // Label: "tls13 " + Label
    std::string full_label = "tls13 " + std::string(label);
    hkdf_label.push_back(static_cast<std::uint8_t>(full_label.size()));
    hkdf_label.insert(hkdf_label.end(), full_label.begin(), full_label.end());
    
    // Context
    hkdf_label.push_back(static_cast<std::uint8_t>(context.size()));
    hkdf_label.insert(hkdf_label.end(), context.begin(), context.end());
    
    // 调用 HKDF-Expand
    return hkdf_expand(secret, hkdf_label, length);
}
```

### Finished 消息验证

```cpp
// 计算 Finished 消息验证数据
std::array<std::uint8_t, 32> compute_finished_verify_data(
    std::span<const std::uint8_t> finished_key,
    std::span<const std::uint8_t> transcript_hash) {
    
    // HMAC-SHA256(finished_key, transcript_hash)
    return hmac_sha256(finished_key, transcript_hash);
}

// 派生 Finished 密钥
std::array<std::uint8_t, 32> derive_finished_key(
    std::span<const std::uint8_t> traffic_secret) {
    
    return hkdf_expand_label(traffic_secret, "finished", {}, 32);
}
```

### 代码位置

| 源文件 | 函数/类 | 用途 |
|--------|---------|------|
| `src/prism/crypto/hkdf.cpp` | `hkdf_extract` | 提取阶段 |
| `src/prism/crypto/hkdf.cpp` | `hkdf_expand` | 扩展阶段 |
| `src/prism/crypto/hkdf.cpp` | `hkdf_expand_label` | TLS 1.3 风格派生 |
| `src/prism/stealth/reality/keygen.cpp` | `derive_handshake_keys` | Reality 握手密钥 |
| `src/prism/stealth/reality/keygen.cpp` | `derive_application_keys` | Reality 应用密钥 |
| `src/prism/stealth/reality/keygen.cpp` | `compute_finished_verify_data` | Finished 验证 |

### 密钥层次结构

Reality/TLS 1.3 的密钥层次：

```
Early Secret
    ↓
Handshake Secret (HKDF-Extract with ECDHE shared secret)
    ├── Client Handshake Traffic Secret
    │       ├── client_key
    │       ├── client_iv
    │       └── client_finished_key
    └── Server Handshake Traffic Secret
            ├── server_key
            ├── server_iv
            └── server_finished_key
    ↓
Master Secret
    ↓
Application Secret
    ├── Client Application Traffic Secret
    │       ├── client_key
    │       └── client_iv
    └── Server Application Traffic Secret
            ├── server_key
            └── server_iv
```

## 安全考量

### 已知安全问题

**弱输入密钥材料**

如果 IKM 的熵极低（如少于安全强度的一半），HKDF 的安全性会降低。提取阶段通过 HMAC 的伪随机性可以部分弥补，但仍建议使用高质量的 IKM。

防御措施：
- 使用高质量的随机源生成 IKM
- 如果 IKM 来自密码，使用 PBKDF2 或 Argon2 进行预处理
- 使用随机盐值增加安全性

**输出长度超限**

超过 255 * HashLen 的输出长度不被支持。请求过长的输出可能导致安全问题。

防御措施：
- 检查输出长度限制
- 对于大量密钥需求，使用多次 HKDF 调用

**盐值重用**

在不同的协议实例中使用相同的盐值和 IKM 会产生相同的 PRK，这可能导致密钥重用问题。

防御措施：
- 每个协议实例使用唯一的盐值
- 盐值可以是协议实例标识符或随机值

**Info 参数冲突**

使用相同的 PRK 和 info 参数会产生相同的 OKM。如果密钥用途没有正确区分，可能导致密钥误用。

防御措施：
- 为每种密钥用途定义唯一的 info 参数
- info 参数应包含用途标识和协议上下文

### 安全建议

1. **盐值使用**
   - 建议总是使用盐值
   - 盐值可以是随机值或协议实例标识
   - 不同协议实例使用不同盐值

2. **Info 参数设计**
   - 包含密钥用途标识
   - 包含协议上下文信息
   - 确保不同密钥用途的 info 不同

3. **输出长度**
   - 不要请求超过实际需求的长度
   - 对于大量密钥，使用结构化派生
   - 注意最大长度限制

4. **密钥存储**
   - PRK 应安全存储
   - 派生完成后，PRK 可以安全删除
   - 输出密钥按用途正确管理

### 安全边界

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 盐值长度 | HashLen | 与哈希输出长度相同 |
| IKM 熵 | ≥HashLen * 8 | 高质量输入 |
| Info 长度 | <HashLen | 保持合理范围 |
| OKM 长度 | ≤255*HashLen | 最大限制 |

### 实现清单

- [ ] 使用推荐的盐值长度
- [ ] 检查 IKM 质量
- [ ] 设计正确的 info 参数
- [ ] 检查输出长度限制
- [ ] 安全存储派生密钥
- [ ] 使用后清理 PRK
- [ ] 正确处理错误情况

## 最佳实践

### 盐值管理

**推荐盐值生成**

```cpp
// 生成 HKDF 盐值
std::vector<std::uint8_t> generate_hkdf_salt(size_t length = 32) {
    std::vector<std::uint8_t> salt(length);
    RAND_bytes(salt.data(), salt.size());
    return salt;
}

// 使用协议上下文作为盐值
std::vector<std::uint8_t> derive_salt_from_context(
    std::span<const std::uint8_t> session_id,
    std::span<const std::uint8_t> server_random) {
    
    std::vector<std::uint8_t> salt;
    salt.insert(salt.end(), session_id.begin(), session_id.end());
    salt.insert(salt.end(), server_random.begin(), server_random.end());
    
    return salt;
}
```

### Info 参数设计

**结构化 Info**

```cpp
// 定义标准的 info 参数结构
struct hkdf_info {
    std::string label;           // 密钥用途标签
    std::vector<std::uint8_t> context;  // 协议上下文
    
    std::vector<std::uint8_t> serialize() const {
        std::vector<std::uint8_t> result;
        
        // 标签长度 + 标签内容
        result.push_back(static_cast<std::uint8_t>(label.size()));
        result.insert(result.end(), label.begin(), label.end());
        
        // 上下文
        result.insert(result.end(), context.begin(), context.end());
        
        return result;
    }
};

// 使用示例
hkdf_info client_key_info{
    .label = "client_encryption_key",
    .context = handshake_hash
};
auto client_key = hkdf_expand(prk, client_key_info.serialize(), 16);
```

### 多密钥派生

**从单个 PRK 派生多个密钥**

```cpp
// TLS 风格的多密钥派生
struct derived_keys {
    std::array<std::uint8_t, 16> encryption_key;
    std::array<std::uint8_t, 12> iv;
    std::array<std::uint8_t, 32> mac_key;
};

derived_keys derive_all_keys(
    std::span<const std::uint8_t> secret,
    std::span<const std::uint8_t> context) {
    
    derived_keys keys;
    
    // 派生加密密钥
    auto enc_secret = hkdf_expand_label(secret, "key", context, 16);
    std::copy(enc_secret.begin(), enc_secret.end(), keys.encryption_key.begin());
    
    // 派生 IV
    auto iv_secret = hkdf_expand_label(secret, "iv", context, 12);
    std::copy(iv_secret.begin(), iv_secret.end(), keys.iv.begin());
    
    // 派生 MAC 密钥
    auto mac_secret = hkdf_expand_label(secret, "mac", context, 32);
    std::copy(mac_secret.begin(), mac_secret.end(), keys.mac_key.begin());
    
    return keys;
}
```

### 错误处理

```cpp
// 安全的 HKDF 调用
std::optional<std::vector<std::uint8_t>> safe_hkdf(
    std::span<const std::uint8_t> salt,
    std::span<const std::uint8_t> ikm,
    std::span<const std::uint8_t> info,
    size_t length) {
    
    // 检查参数
    if (length > 255 * 32) {  // SHA-256 最大输出
        return std::nullopt;
    }
    
    if (ikm.empty()) {
        return std::nullopt;
    }
    
    try {
        return hkdf_sha256(salt, ikm, info, length);
    } catch (...) {
        return std::nullopt;
    }
}
```

### 密钥清理

```cpp
// 安全的密钥派生上下文
class hkdf_context {
public:
    hkdf_context(
        std::span<const std::uint8_t> salt,
        std::span<const std::uint8_t> ikm) {
        
        prk_ = hkdf_extract(salt, ikm);
    }
    
    ~hkdf_context() {
        // 清理 PRK
        OPENSSL_cleanse(prk_.data(), prk_.size());
    }
    
    std::vector<std::uint8_t> expand(
        std::span<const std::uint8_t> info,
        size_t length) {
        
        return hkdf_expand(prk_, info, length);
    }
    
private:
    std::vector<std::uint8_t> prk_;
};
```

## 常见问题

### Q1: HKDF 的盐值是必须的吗？

盐值是可选的，但强烈推荐使用。没有盐值时，HKDF 使用全零作为默认盐值。使用盐值可以：
- 增加密钥派生的随机性
- 支持从同一 IKM 派生不同的 PRK
- 提高对弱 IKM 的安全性

### Q2: HKDF 的最大输出长度是多少？

最大输出长度为 255 * HashLen 字节：
- HKDF-SHA256：最大 8160 字节（255 * 32）
- HKDF-SHA384：最大 12240 字节（255 * 48）
- HKDF-SHA512：最大 16320 字节（255 * 64）

超过这个长度需要多次调用 HKDF。

### Q3: Info 参数有什么作用？

Info 参数用于：
- 区分从同一 PRK 派生的不同密钥
- 将密钥绑定到特定用途
- 包含协议上下文信息

正确使用 Info 参数可以防止密钥误用和交叉攻击。

### Q4: HKDF-SHA256 和 HKDF-SHA384 如何选择？

选择建议：
- HKDF-SHA256：大多数场景的默认选择
- HKDF-SHA384：需要更高安全强度的场景
- HKDF-SHA512：最高安全需求，但性能略低

TLS 1.3 根据密码套件选择相应的 HKDF。

### Q5: HKDF 可以用于密码派生吗？

不推荐直接使用。密码熵通常较低，HKDF 的提取阶段虽然可以处理弱 IKM，但不如专门设计的密码哈希函数安全。推荐使用：
- PBKDF2：NIST 标准的密码派生
- Argon2：现代密码哈希竞赛优胜者
- scrypt：内存密集型密码哈希

如果必须使用 HKDF 派生密码密钥，建议先用 PBKDF2/Argon2 预处理密码。

### Q6: HKDF-Extract 和 HKDF-Expand 可以单独使用吗？

可以单独使用：
- HKDF-Extract：用于随机性提取，生成 PRK
- HKDF-Expand：用于密钥扩展，从已有的 PRK 派生更多密钥

在 TLS 1.3 中，Extract 和 Expand 都有独立的用途。

### Q7: PRK 可以直接作为密钥使用吗？

理论上可以，但需要谨慎：
- PRK 长度等于 HashLen，可能不符合某些密钥长度要求
- 直接使用 PRK 可能破坏密钥分离原则
- 推荐通过 Expand 派生具体的密钥

### Q8: TLS 1.3 为什么使用 HKDF？

TLS 1.3 使用 HKDF 的原因：
- 标准化的密钥派生方法
- 清晰的密钥层次结构
- 支持密钥分离和绑定
- 经过充分的安全性证明
- 实现简单，易于验证

### Q9: HKDF 与其他 KDF 如何比较？

与其他 KDF 的比较：

| KDF | 特点 | 适用场景 |
|-----|------|----------|
| HKDF | 两阶段设计，灵活 | 通用密钥派生 |
| PBKDF2 | 迭代慢哈希 | 密码派生 |
| Argon2 | 内存密集 | 密码哈希 |
| KDF in SP 800-56C | 专门设计 | 密钥协商 |

HKDF 是最通用的 KDF，适合从高质量密钥材料派生密钥。

### Q10: HKDF 的性能如何？

HKDF 的性能取决于底层 HMAC：
- 单次 HMAC-SHA256 约 5-10 微秒
- HKDF-Extract：1 次 HMAC
- HKDF-Expand：ceil(L/32) 次 HMAC

对于 32 字节输出，HKDF 约需要 2 次 HMAC 操作，总耗时约 10-20 微秒。