---
title: "ECDHE"
category: "crypto"
type: ref
module: ref
source: "RFC 4492, RFC 8422"
tags: [密码学, ecdhe, 密钥交换, 前向保密]
created: 2026-05-15
updated: 2026-05-17
---

# ECDHE

**类别**: 密码学

## 概述

ECDHE（Elliptic Curve Diffie-Hellman Ephemeral）是基于椭圆曲线的临时 Diffie-Hellman 密钥交换协议。ECDHE 是现代密码学中最重要的密钥交换机制之一，广泛用于 TLS 1.2、TLS 1.3、SSH、Signal、WireGuard 等安全通信协议中。

ECDHE 的核心特性是前向保密（Forward Secrecy）。前向保密确保即使长期私钥在未来被泄露，历史会话的加密数据也无法被解密。这是通过使用临时密钥实现的：每次会话都生成新的临时密钥对，会话结束后临时私钥被销毁，使得历史会话的共享密钥无法被恢复。

ECDHE 结合了椭圆曲线密码学（ECC）的高效性和临时密钥交换的安全性。与传统 RSA 密钥交换相比，ECDHE 提供了更高的安全保证（前向保密）、更小的密钥尺寸（256 位 vs 2048 位）和更高的计算效率（约 10 倍）。这些优势使得 ECDHE 成为现代 TLS 协议的首选密钥交换方式。

椭圆曲线密码学基于椭圆曲线离散对数问题（ECDLP）的数学难度。在椭圆曲线上，给定基点 G 和点 P = nG，求解 n 是计算上不可行的。这个问题的难度远高于整数离散对数问题，使得 ECC 可以使用更小的密钥达到相同的安全强度。例如，256 位的椭圆曲线密钥与 3072 位的 RSA 密钥具有相同的安全强度。

ECDHE 在 TLS 1.2 中是可选的，但在 TLS 1.3 中成为强制要求。TLS 1.3 完全移除了静态 RSA 密钥交换，所有连接都必须使用 ECDHE（或其他提供前向保密的密钥交换方式）。这一设计决策反映了密码学社区对前向保密重要性的共识。

TLS 1.3 支持多种 ECDHE 曲线组：
- x25519（Curve25519）：强制支持，推荐使用
- secp256r1（P-256）：可选，NIST 标准曲线
- secp384r1（P-384）：可选，更高安全强度
- secp521r1（P-521）：可选，最高安全强度
- x448（Curve448）：可选，224 位安全强度

x25519 是 TLS 1.3 的默认选择，因为它提供了更好的安全性保证、更高的性能和更简单的实现。

ECDHE 的工作流程如下：
1. 客户端和服务端各自生成临时密钥对
2. 客户端在 ClientHello 中发送临时公钥
3. 服务端在 ServerHello 中发送临时公钥
4. 双方使用对方的临时公钥和自己的临时私钥计算共享密钥
5. 使用 HKDF 从共享密钥派生会话加密密钥
6. 会话结束后销毁临时私钥

前向保密的安全性依赖于临时私钥的正确销毁。如果临时私钥在会话结束后仍然存在（例如存储在内存或日志中），前向保密就会失效。因此，实现 ECDHE 时必须确保临时私钥在使用后被安全清零。

Signal 协议使用 ECDHE 的变体实现 Double Ratchet 算法。每次消息发送都会生成新的临时密钥对，提供消息级别的前向保密和未来保密。这种设计比 TLS 的会话级别前向保密更强。

WireGuard VPN 使用 ECDHE 作为唯一的密钥交换方式。WireGuard 的 Noise Framework 协议使用长期密钥和临时密钥的组合，通过 ECDHE 每分钟轮换会话密钥，提供持续的前向保密。

在代理服务器和网络协议实现中，ECDHE 是建立安全连接的基础。Prism 在 Reality 协议中使用 ECDHE（通过 X25519）进行密钥协商，然后通过 HKDF 派生出会话加密密钥。ECDHE 提供了高效、安全的密钥交换基础，同时确保历史连接的前向保密。

总的来说，ECDHE 代表了现代密钥交换的最佳实践。它通过椭圆曲线的高效性提供了快速密钥交换，通过临时密钥确保了前向保密，成为 TLS 1.3 和现代安全协议的核心组件。理解 ECDHE 的工作原理和正确实现方法对于构建安全的通信系统至关重要。

## 算法原理

### Diffie-Hellman 密钥交换

Diffie-Hellman（DH）是最早提出的密钥交换协议，由 Whitfield Diffie 和 Martin Hellman 在 1976 年发表。

**基本 DH 算法**

在有限域 GF(p) 上：
1. 选择素数 p 和基点 g
2. Alice 选择私钥 a，计算公钥 A = g^a mod p
3. Bob 选择私钥 b，计算公钥 B = g^b mod p
4. Alice 计算共享密钥：K = B^a mod p = g^ab mod p
5. Bob 计算共享密钥：K = A^b mod p = g^ab mod p

安全性基于离散对数问题：从 A = g^a 求解 a 是计算上不可行的。

**DH 的局限性**

传统 DH 的局限性：
- 需要大素数（至少 2048 位）保证安全性
- 计算效率较低（模幂运算）
- 不提供前向保密（如果是静态密钥）
- 实现复杂，容易出现漏洞

### 椭圆曲线密码学

椭圆曲线密码学（ECC）使用椭圆曲线上的点运算替代有限域上的模幂运算。

**椭圆曲线定义**

椭圆曲线是满足以下方程的点集合：

```
y² = x³ + ax + b (mod p)
```

加上一个"无穷远点"O。

对于不同形式的曲线，方程有所不同：
- Weierstrass 形式：y² = x³ + ax + b
- Montgomery 形式：By² = x³ + Ax² + x
- Edwards 形式：x² + y² = 1 + dx²y²

**点运算**

椭圆曲线上的点运算：
- 点加法：P + Q，几何上通过弦线与曲线的交点
- 点倍乘：P + P = 2P，几何上通过切线与曲线的交点
- 点乘法：nP = P + P + ... + P（n 次）

点乘法是 ECC 的核心运算，类似于 DH 中的模幂运算。

**椭圆曲线离散对数问题**

给定基点 G 和点 P = nG，求解 n 是计算上不可行的。这是 ECC 安全性的数学基础。

### ECDHE 算法

ECDHE 将 ECDH（椭圆曲线 DH）与临时密钥结合。

**基本 ECDHE 算法**

1. Alice 生成临时密钥对 (a, A = aG)
2. Bob 生成临时密钥对 (b, B = bG)
3. Alice 发送临时公钥 A 给 Bob
4. Bob 发送临时公钥 B 给 Alice
5. Alice 计算共享密钥：K = aB = abG
6. Bob 计算共享密钥：K = bA = abG
7. 双方使用 K 派生会话密钥
8. Alice 和 Bob 销毁临时私钥 a 和 b

**前向保密保证**

即使长期私钥在未来泄露：
- 临时私钥 a 和 b 已被销毁
- 无法从 A 和 B 恢复 K（ECDLP 困难）
- 历史会话数据无法解密

### TLS 1.3 ECDHE 流程

TLS 1.3 使用 ECDHE 作为核心密钥交换机制。

**握手流程**

```
Client                                    Server
  |                                         |
  | ClientHello                             |
  |   + key_share (client_epk)              |
  |---------------------------------------->|
  |                                         |
  |                          ServerHello    |
  |                   + key_share (server_epk)|
  |<----------------------------------------|
  |                                         |
  | EncryptedExtensions                     |
  |<----------------------------------------|
  |                                         |
  | Certificate                             |
  |<----------------------------------------|
  |                                         |
  | CertificateVerify                       |
  |<----------------------------------------|
  |                                         |
  | Finished                                |
  |<----------------------------------------|
  |                                         |
  | Finished                                |
  |---------------------------------------->|
  |                                         |
```

**密钥派生层次**

```
Early Secret (可选)
    ↓
Handshake Secret = HKDF-Extract(EarlySecret, ECDHE_SharedSecret)
    ↓
Client Handshake Traffic Secret
Server Handshake Traffic Secret
    ↓
Master Secret = HKDF-Extract(HandshakeSecret, 0)
    ↓
Client Application Traffic Secret
Server Application Traffic Secret
```

**Key Share 扩展**

Key Share 扩展包含客户端的临时公钥：

```
struct {
    NamedGroup named_group;
    opaque key_exchange<1..2^16-1>;
} KeyShareEntry;

struct {
    KeyShareEntry client_shares<0..2^16-1>;
} KeyShareClientHello;
```

客户端可以发送多个 KeyShareEntry（不同曲线组），服务端选择其中一个。

### 不同曲线的实现

**X25519**

使用 Montgomery ladder 点乘，恒定时间，高性能：

```cpp
// X25519 ECDHE
class x25519_ecdhe {
public:
    x25519_ecdhe() {
        private_key_ = generate_x25519_private_key();
        public_key_ = compute_x25519_public_key(private_key_);
    }
    
    auto get_public_key() const { return public_key_; }
    
    auto compute_shared_secret(std::span<const std::uint8_t, 32> peer_public) {
        return x25519_scalar_multiply(private_key_, peer_public);
    }
    
    ~x25519_ecdhe() {
        OPENSSL_cleanse(private_key_.data(), 32);
    }
    
private:
    std::array<std::uint8_t, 32> private_key_;
    std::array<std::uint8_t, 32> public_key_;
};
```

**P-256 (secp256r1)**

使用 Weierstrass 形式曲线，需要公钥验证：

```cpp
// P-256 ECDHE
class p256_ecdhe {
public:
    p256_ecdhe() {
        EC_GROUP* group = EC_GROUP_new_by_curve_name(NID_X9_62_prime256v1);
        EC_KEY* key = EC_KEY_new();
        EC_KEY_generate_key(key);
        
        // 提取私钥和公钥
        private_key_ = extract_private_key(key);
        public_key_ = extract_public_key(key);
        
        EC_KEY_free(key);
        EC_GROUP_free(group);
    }
    
    auto compute_shared_secret(std::span<const std::uint8_t> peer_public) {
        // 验证公钥在曲线上
        if (!validate_public_key(peer_public)) {
            throw std::runtime_error("Invalid peer public key");
        }
        
        return compute_ecdh_shared_secret(private_key_, peer_public);
    }
    
private:
    std::vector<std::uint8_t> private_key_;
    std::vector<std::uint8_t> public_key_;
};
```

### 安全性分析

**前向保密安全性**

前向保密的安全性依赖于：
1. 临时私钥的正确销毁
2. ECDLP 的计算难度
3. 密钥派生函数的安全性

如果临时私钥泄露，前向保密失效。

**椭圆曲线安全强度**

不同曲线的安全强度：

| 曲线 | 密钥长度 | 安全强度 |
|------|----------|----------|
| X25519 | 256 位 | 128 位 |
| P-256 | 256 位 | 128 位 |
| P-384 | 384 位 | 192 位 |
| P-521 | 521 位 | 256 位 |
| X448 | 448 位 | 224 位 |

**已知攻击**

椭圆曲线的已知攻击：
- Pollard's rho：约 O(√n) 次操作，是最佳攻击
- 小子群攻击：使用 cofactor 处理防御
- 无效曲线攻击：需要公钥验证防御

X25519 通过 clamping 天然防御这些攻击，P-256 需要显式验证。

### 数学基础

**椭圆曲线点加法**

对于 Weierstrass 形式曲线 y² = x³ + ax + b：

点加法 P + Q = R：
```
如果 P ≠ Q：
    s = (yQ - yP) / (xQ - xP) mod p
    xR = s² - xP - xQ mod p
    yR = s(xP - xR) - yP mod p

如果 P = Q（点倍乘）：
    s = (3xP² + a) / (2yP) mod p
    xR = s² - 2xP mod p
    yR = s(xP - xR) - yP mod p
```

**点乘法算法**

点乘法 nG 的实现：

```
算法 DoubleAndAdd(G, n):
    result = O（无穷远点）
    temp = G
    
    while n > 0:
        if n & 1:
            result = result + temp
        temp = temp + temp（倍乘）
        n = n >> 1
    
    return result
```

**Montgomery Ladder**

对于 Montgomery 形式曲线，使用更高效的 ladder 算法：

```
算法 MontgomeryLadder(u, n):
    x1 = u, x2 = 1, z2 = 0, x3 = u, z3 = 1
    
    for i from 254 downto 0:
        // 条件交换
        swap = (n >> i) & 1
        cswap(swap, x2, x3)
        cswap(swap, z2, z3)
        
        // Double and add
        // ...（详细计算）
    
    return x2 / z2
```

## 在 Prism 中的应用

Prism 在 Reality 协议中使用 ECDHE 进行密钥交换。

### Reality 密钥交换

Reality 协议使用 X25519 实现 ECDHE。

**服务端密钥配置**

```cpp
// Reality 服务端配置
struct reality_server_config {
    std::array<std::uint8_t, 32> private_key;   // 长期私钥
    std::array<std::uint8_t, 32> public_key;    // 长期公钥
    std::vector<std::string> server_names;
    std::vector<std::array<std::uint8_t, 8>> short_ids;
};

// 生成 Reality 密钥对
reality_server_config generate_reality_config() {
    reality_server_config config;
    
    // 生成 X25519 密钥对
    config.private_key = generate_x25519_private_key();
    config.public_key = compute_x25519_public_key(config.private_key);
    
    return config;
}
```

**客户端握手**

```cpp
// Reality 客户端 ECDHE 握手
class reality_client_handshake {
public:
    reality_client_handshake(const reality_server_config& config) 
        : server_config_(config) {
        
        // 生成临时密钥对
        ephemeral_private_ = generate_x25519_private_key();
        ephemeral_public_ = compute_x25519_public_key(ephemeral_private_);
    }
    
    auto build_client_hello() -> std::vector<std::uint8_t> {
        // 构建 ClientHello，包含临时公钥
        // ...
    }
    
    auto process_server_response(
        std::span<const std::uint8_t> server_response) 
        -> session_keys {
        
        // 提取服务端临时公钥
        auto server_ephemeral_public = extract_server_ephemeral_public(server_response);
        
        // 计算临时共享密钥
        auto ephemeral_shared = x25519_scalar_multiply(
            ephemeral_private_, server_ephemeral_public);
        
        // 使用 HKDF 派生会话密钥
        return derive_session_keys(ephemeral_shared);
    }
    
    ~reality_client_handshake() {
        // 销毁临时私钥（前向保密）
        OPENSSL_cleanse(ephemeral_private_.data(), 32);
    }
    
private:
    reality_server_config server_config_;
    std::array<std::uint8_t, 32> ephemeral_private_;
    std::array<std::uint8_t, 32> ephemeral_public_;
};
```

**密钥派生**

```cpp
// Reality 密钥派生
session_keys derive_session_keys(
    std::span<const std::uint8_t, 32> shared_secret) {
    
    session_keys keys;
    
    // 提取伪随机密钥
    auto early_secret = hkdf_extract({}, {});
    auto handshake_secret = hkdf_extract(early_secret, shared_secret);
    
    // 派生客户端密钥
    auto client_secret = hkdf_expand_label(handshake_secret, "c hs traffic", {}, 32);
    keys.client_encryption_key = hkdf_expand_label(client_secret, "key", {}, 16);
    keys.client_iv = hkdf_expand_label(client_secret, "iv", {}, 12);
    
    // 派生服务端密钥
    auto server_secret = hkdf_expand_label(handshake_secret, "s hs traffic", {}, 32);
    keys.server_encryption_key = hkdf_expand_label(server_secret, "key", {}, 16);
    keys.server_iv = hkdf_expand_label(server_secret, "iv", {}, 12);
    
    return keys;
}
```

### 代码位置

| 源文件 | 函数/类 | 用途 |
|--------|---------|------|
| `src/prism/crypto/x25519.cpp` | `x25519_keypair_generate` | 生成密钥对 |
| `src/prism/crypto/x25519.cpp` | `x25519_shared_secret` | 计算共享密钥 |
| `src/prism/stealth/reality/handshake.cpp` | `reality_handshake` | Reality 握手 |
| `src/prism/stealth/reality/keygen.cpp` | `derive_handshake_keys` | 密钥派生 |

### ECDHE 生命周期

```cpp
// ECDHE 会话生命周期管理
class ecdhe_session {
public:
    // 阶段 1：生成临时密钥
    void generate_ephemeral_keys() {
        ephemeral_private_ = generate_x25519_private_key();
        ephemeral_public_ = compute_x25519_public_key(ephemeral_private_);
    }
    
    // 阶段 2：计算共享密钥
    void compute_shared_secret(
        std::span<const std::uint8_t, 32> peer_public) {
        
        shared_secret_ = x25519_scalar_multiply(ephemeral_private_, peer_public);
    }
    
    // 阶段 3：派生会话密钥
    void derive_session_keys() {
        session_keys_ = derive_session_keys(shared_secret_);
    }
    
    // 阶段 4：销毁临时私钥
    void destroy_ephemeral_private() {
        OPENSSL_cleanse(ephemeral_private_.data(), 32);
        ephemeral_private_.fill(0);  // 标记为已销毁
    }
    
    // 阶段 5：会话结束，销毁所有密钥
    void end_session() {
        OPENSSL_cleanse(shared_secret_.data(), 32);
        OPENSSL_cleanse(session_keys_.client_encryption_key.data(), 16);
        OPENSSL_cleanse(session_keys_.server_encryption_key.data(), 16);
        // ...
    }
    
private:
    std::array<std::uint8_t, 32> ephemeral_private_;
    std::array<std::uint8_t, 32> ephemeral_public_;
    std::array<std::uint8_t, 32> shared_secret_;
    session_keys session_keys_;
};
```

### 密钥轮换

Reality 支持密钥轮换，保持前向保密：

```cpp
// 密钥轮换机制
class key_rotation_manager {
public:
    void rotate_if_needed() {
        auto now = std::chrono::system_clock::now();
        auto age = now - current_key_.created_at;
        
        // 基于时间轮换（例如 10 分钟）
        if (age > std::chrono::minutes(10)) {
            rotate_key();
        }
        
        // 基于数据量轮换
        if (current_key_.bytes_encrypted > MAX_BYTES_PER_KEY) {
            rotate_key();
        }
    }
    
private:
    void rotate_key() {
        // 销毁旧密钥
        destroy_old_session_keys();
        
        // 生成新的临时密钥
        generate_new_ephemeral_keys();
        
        // 重新协商
        renegotiate_session_keys();
    }
    
    static constexpr size_t MAX_BYTES_PER_KEY = 1ULL << 30;  // 1 GB
};
```

## 安全考量

### 已知安全问题

**临时私钥泄露**

如果临时私钥在会话结束后仍然存在，前向保密失效。

防御措施：
- 使用后立即销毁临时私钥
- 使用安全内存清理（OPENSSL_cleanse）
- 避免将密钥写入日志或磁盘

**小子群攻击**

某些曲线有小子群（cofactor > 1），对手可能使用小子群元素作为公钥。

防御措施：
- X25519：私钥 clamping 确保点乘结果在主子群
- P-256：验证公钥在正确子群内

**无效曲线攻击**

对于 Weierstrass 形式曲线，对手可能使用不在曲线上的点。

防御措施：
- 验证公钥在曲线上
- 使用 Montgomery 形式曲线（X25519）

**重放攻击**

ECDHE 本身不防止重放，需要额外的机制。

防御措施：
- 使用会话标识符
- 包含时间戳
- 使用序列号

### 安全建议

1. **密钥管理**
   - 使用后立即销毁临时私钥
   - 使用安全内存清理
   - 定期轮换密钥

2. **曲线选择**
   - 优先使用 X25519
   - 避免 NIST 曲线的潜在问题
   - 使用安全强度足够的曲线

3. **实现验证**
   - 使用经过验证的加密库
   - 测试恒定时间执行
   - 测试特殊输入处理

4. **密钥派生**
   - 使用 HKDF 派生密钥
   - 包含协议上下文信息
   - 确保密钥分离

### 安全边界

| 属性 | X25519 | P-256 |
|------|--------|-------|
| 安全强度 | 128 位 | 128 位 |
| Cofactor | 8 | 1 |
| 公钥验证 | 不需要 | 需要 |
| 恒定时间 | 天然 | 需要实现 |

### 实现清单

- [ ] 生成安全的临时密钥
- [ ] 使用后销毁临时私钥
- [ ] 正确的密钥派生（HKDF）
- [ ] 恒定时间点乘
- [ ] 公钥验证（P-256）
- [ ] 定期密钥轮换
- [ ] 安全内存清理

## 最佳实践

### 临时密钥管理

**安全的临时密钥生命周期**

```cpp
// 临时密钥的完整生命周期管理
class ephemeral_key_manager {
public:
    ephemeral_key_manager() {
        // 使用安全随机源
        ephemeral_private_ = generate_safe_ephemeral_private();
        ephemeral_public_ = compute_public(ephemeral_private_);
    }
    
    auto negotiate(std::span<const std::uint8_t> peer_public) 
        -> negotiated_result {
        
        // 计算共享密钥
        auto shared = compute_shared(ephemeral_private_, peer_public);
        
        // 派生会话密钥
        auto session = derive_session(shared);
        
        // 立即销毁临时私钥
        destroy_ephemeral();
        
        return session;
    }
    
    ~ephemeral_key_manager() {
        // 确保临时私钥被销毁
        destroy_ephemeral();
    }
    
private:
    void destroy_ephemeral() {
        // 恒定时间清零
        OPENSSL_cleanse(ephemeral_private_.data(), ephemeral_private_.size());
        ephemeral_destroyed_ = true;
    }
    
    std::array<std::uint8_t, 32> ephemeral_private_;
    std::array<std::uint8_t, 32> ephemeral_public_;
    bool ephemeral_destroyed_ = false;
};
```

### 密钥派生

**标准密钥派生流程**

```cpp
// ECDHE 后的密钥派生
struct all_session_keys {
    std::array<std::uint8_t, 16> client_enc_key;
    std::array<std::uint8_t, 12> client_enc_iv;
    std::array<std::uint8_t, 16> server_enc_key;
    std::array<std::uint8_t, 12> server_enc_iv;
    std::array<std::uint8_t, 32> client_mac_key;
    std::array<std::uint8_t, 32> server_mac_key;
};

all_session_keys derive_all_keys(
    std::span<const std::uint8_t, 32> shared_secret,
    std::span<const std::uint8_t> handshake_hash) {
    
    all_session_keys keys;
    
    // 提取伪随机密钥
    auto secret = hkdf_extract({}, shared_secret);
    
    // 派生客户端加密密钥
    auto client_traffic = hkdf_expand_label(secret, "c ap traffic", handshake_hash, 32);
    keys.client_enc_key = hkdf_expand_label(client_traffic, "key", {}, 16);
    keys.client_enc_iv = hkdf_expand_label(client_traffic, "iv", {}, 12);
    
    // 派生服务端加密密钥
    auto server_traffic = hkdf_expand_label(secret, "s ap traffic", handshake_hash, 32);
    keys.server_enc_key = hkdf_expand_label(server_traffic, "key", {}, 16);
    keys.server_enc_iv = hkdf_expand_label(server_traffic, "iv", {}, 12);
    
    return keys;
}
```

### 密钥轮换策略

**自动密钥轮换**

```cpp
// 密钥轮换策略
class key_rotation_policy {
public:
    key_rotation_policy() 
        : rotation_interval_(std::chrono::minutes(10)),
          max_bytes_per_key_(1ULL << 30),  // 1 GB
          max_messages_per_key_(1ULL << 20) {}  // 1M
    
    bool should_rotate(const session_stats& stats) const {
        // 时间检查
        auto age = std::chrono::system_clock::now() - stats.key_created;
        if (age > rotation_interval_) {
            return true;
        }
        
        // 数据量检查
        if (stats.bytes_sent + stats.bytes_received > max_bytes_per_key_) {
            return true;
        }
        
        // 消息数检查
        if (stats.messages_sent > max_messages_per_key_) {
            return true;
        }
        
        return false;
    }
    
    void perform_rotation(session_context& ctx) {
        // 销毁旧密钥
        ctx.destroy_current_keys();
        
        // 发起新的 ECDHE
        ctx.initiate_new_key_exchange();
        
        // 更新统计
        ctx.reset_stats();
    }
    
private:
    std::chrono::minutes rotation_interval_;
    size_t max_bytes_per_key_;
    size_t max_messages_per_key_;
};
```

### 多曲线支持

**曲线协商**

```cpp
// 多曲线协商
enum class ecdhe_curve {
    x25519,
    p256,
    p384,
    x448
};

class ecdhe_negotiator {
public:
    // 客户端发送支持的曲线
    std::vector<std::uint8_t> build_key_share_offer() {
        std::vector<std::uint8_t> offer;
        
        // 按优先级发送多个曲线的 key share
        for (auto curve : preferred_curves_) {
            auto share = generate_key_share(curve);
            append_key_share(offer, curve, share);
        }
        
        return offer;
    }
    
    // 服务端选择曲线
    ecdhe_context accept_key_share(
        std::span<const std::uint8_t> client_offer) {
        
        // 解析客户端支持的曲线
        auto client_curves = parse_supported_curves(client_offer);
        
        // 选择服务端支持的最高优先级曲线
        ecdhe_curve selected = select_curve(client_curves, supported_curves_);
        
        // 创建对应曲线的 ECDHE 上下文
        return create_ecdhe_context(selected);
    }
    
private:
    std::vector<ecdhe_curve> preferred_curves_ = {
        ecdhe_curve::x25519,
        ecdhe_curve::p384
    };
    std::vector<ecdhe_curve> supported_curves_ = {
        ecdhe_curve::x25519,
        ecdhe_curve::p256,
        ecdhe_curve::p384
    };
};
```

## 常见问题

### Q1: ECDHE 与静态 ECDH 有什么区别？

静态 ECDH 使用长期密钥，不提供前向保密：
- 期私钥泄露后，历史会话可被解密
- 密钥泄露的影响是永久的

ECDHE 使用临时密钥，提供前向保密：
- 临时私钥在使用后销毁
- 即使长期密钥泄露，历史会话仍安全

TLS 1.3 强制使用 ECDHE，移除了静态 ECDH。

### Q2: 为什么 TLS 1.3 强制使用 ECDHE？

TLS 1.3 强制 ECDHE 的原因：
- 前向保密是现代密码学的基本要求
- 避免 RSA 密钥交换的各种问题
- 统一密钥派生流程
- 减少协议复杂性

### Q3: X25519 与 P-256 如何选择？

X25519 相比 P-256 的优势：
- 更强的安全性保证（无 NSA 参与）
- 更高的性能（约 10 倍）
- 恒定时间天然保证
- 无需公钥验证
- 更简单的实现

TLS 1.3 将 X25519 作为默认和强制支持的曲线。

### Q4: ECDHE 的前向保密如何保证？

前向保密依赖于：
1. 临时私钥的正确销毁
2. ECDLP 的计算难度
3. 密钥派生函数的安全性

如果临时私钥泄露，前向保密失效。必须确保临时私钥在使用后被安全清零。

### Q5: ECDHE 可以与长期密钥结合吗？

是的。TLS 1.3 和 Signal 协议使用长期密钥和临时密钥的组合：
- 长期密钥用于身份验证
- 临时密钥用于密钥交换
- 两者结合提供认证和前向保密

Signal 的 Double Ratchet 进一步扩展了这一设计。

### Q6: ECDHE 的密钥尺寸是多少？

不同曲线的密钥尺寸：
- X25519：32 字节私钥 + 32 字节公钥
- P-256：32 字节私钥 + 64 字节公钥（未压缩）
- P-384：48 字节私钥 + 96 字节公钥（未压缩）

X25519 的密钥尺寸最小，传输效率最高。

### Q7: ECDHE 需要公钥验证吗？

取决于曲线类型：
- X25519：不需要显式验证（clamping 提供保护）
- P-256：需要验证公钥在曲线上

现代实现应优先使用 X25519 以避免验证复杂性。

### Q8: ECDHE 在嵌入式设备上性能如何？

ECDHE 在嵌入式设备上的性能：
- ARM Cortex-M4：单次 X25519 约 100-200 ms
- ARM Cortex-M0：单次 X25519 约 1-2 s

对于低功耗场景，可以预计算或使用更快的曲线。

### Q9: Reality 协议为什么使用 ECDHE？

Reality 使用 ECDHE 的原因：
- 提供前向保密
- 高效的密钥交换
- 与 TLS 1.3 兼容
- X25519 的安全性保证
- 支持密钥轮换

### Q10: ECDHE 的安全强度是多少？

不同曲线的安全强度：
- X25519：128 位（与 AES-128 相当）
- P-256：128 位
- P-384：192 位
- X448：224 位

128 位安全强度对于绝大多数应用已经足够