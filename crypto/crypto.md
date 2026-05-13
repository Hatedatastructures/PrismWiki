---
title: Crypto 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [crypto, security, encryption, infrastructure]
related: [[stealth/overview], [fault], [pipeline/overview], [architecture]]
module: crypto
category: security
layer: infrastructure
status: stable
dependencies:
  - BoringSSL
  - BLAKE3
  - fault
dependents:
  - stealth
  - pipeline
  - protocol
---

# Crypto 模块

## 1. 模块定位

Crypto 模块为 Prism 提供**统一的密码学原语层**，封装 BoringSSL 和 BLAKE3 的底层 API，为上层协议（SS2022、Reality、Trojan、TLS 1.3）提供类型安全的 C++ 接口。

**核心价值**：将分散的密码学操作统一到 `psm::crypto` 命名空间，隔离 BoringSSL 头文件污染，提供一致的错误处理模式。

**边界**：Crypto 模块不包含协议逻辑，仅提供原语。协议层在 [[pipeline]] 和 [[stealth]] 中使用这些原语。

## 2. 核心功能

### 2.1 AEAD 加解密
支持 AES-128/256-GCM、ChaCha20-Poly1305、XChaCha20-Poly1305 四种算法。`aead_context` 管理 nonce 自动递增（小端序），同时提供显式 nonce 重载用于 UDP 逐包加密。

### 2.2 密钥派生
- **BLAKE3 derive_key**：SS2022 会话密钥派生，基于上下文字符串实现域分离
- **HKDF-SHA256**：TLS 1.3 密钥调度核心，含 HMAC-SHA256/512、HKDF-Extract/Expand、HKDF-Expand-Label

### 2.3 密钥交换
- **X25519**：Curve25519 ECDHE，用于 Reality 协议的密钥交换（128 位安全强度）

### 2.4 辅助工具
- **AES-ECB 单块**：SS2022 UDP SeparateHeader 加密（16 字节块）
- **Base64 编解码**：HTTP Basic 认证等场景，支持 URL-safe 变体
- **SHA-224**：Trojan 协议密码哈希，含凭据规范化
- **SHA-256**：TLS 1.3 transcript hash，支持多块拼接

## 3. 算法支持矩阵

| 算法 | 密钥长度 | Nonce 长度 | Tag 长度 | 用途 |
|------|---------|-----------|---------|------|
| AES-128-GCM | 16B | 12B | 16B | SS2022 TCP |
| AES-256-GCM | 32B | 12B | 16B | SS2022 TCP |
| ChaCha20-Poly1305 | 32B | 12B | 16B | SS2022 TCP |
| XChaCha20-Poly1305 | 32B | 24B | 16B | SS2022 UDP |
| AES-ECB | 16/32B | — | — | SS2022 UDP Header |
| X25519 | 32B | — | — | Reality ECDHE |
| BLAKE3 | — | — | — | SS2022 密钥派生 |
| HKDF-SHA256 | — | — | — | TLS 1.3 密钥调度 |

## 4. 关键类和接口

### 4.1 `aead_context` — AEAD 加密上下文（`aead.hpp`）

```cpp
enum class aead_cipher : std::uint8_t {
    aes_128_gcm, aes_256_gcm, chacha20_poly1305, xchacha20_poly1305
};

class aead_context {
public:
    explicit aead_context(aead_cipher cipher, std::span<const std::uint8_t> key);
    ~aead_context();
    // 不可拷贝，可移动

    auto seal(out, plaintext, ad = {}) -> fault::code;  // 自动递增 nonce（小端序）
    auto open(out, ciphertext, ad = {}) -> fault::code;
    auto seal(out, plaintext, nonce, ad) -> fault::code;  // 显式 nonce（用于 UDP）
    auto open(out, ciphertext, nonce, ad) -> fault::code;

    [[nodiscard]] static constexpr auto tag_length() -> std::size_t { return 16; }
    [[nodiscard]] auto nonce_length() const -> std::size_t;  // 12 或 24
    [[nodiscard]] static constexpr auto seal_output_size(len) -> std::size_t;
    [[nodiscard]] static constexpr auto open_output_size(len) -> std::size_t;
};
```

前向声明 `evp_aead_ctx_st` 避免暴露 OpenSSL 头文件。nonce 按小端序自动递增，符合 SS2022 规范。

### 4.2 BLAKE3 密钥派生（`blake3.hpp`）

```cpp
namespace psm::crypto {
    auto derive_key(std::string_view context, std::span<const std::uint8_t> material,
                    std::size_t out_len, std::span<std::uint8_t> out) -> void;
    [[nodiscard]] auto derive_key(std::string_view context, std::span<const std::uint8_t> material,
                                   std::size_t out_len) -> std::vector<std::uint8_t>;
}
```

上下文字符串用于域分离，SS2022 使用 `"shadowsocks 2022 session subkey"`。

### 4.3 X25519 密钥交换（`x25519.hpp`）

```cpp
constexpr std::size_t X25519_KEY_LEN = 32;
struct x25519_keypair {
    std::array<std::uint8_t, 32> private_key{};
    std::array<std::uint8_t, 32> public_key{};
};
[[nodiscard]] auto generate_x25519_keypair() -> x25519_keypair;
[[nodiscard]] auto derive_x25519_public_key(private_key) -> std::array<..., 32>;
auto x25519(private_key, peer_public_key) -> std::pair<fault::code, std::array<..., 32>>;
```

### 4.4 HKDF 密钥派生（`hkdf.hpp`）

```cpp
[[nodiscard]] auto hmac_sha256(key, data) -> std::array<uint8_t, 32>;
[[nodiscard]] auto hmac_sha512(key, data) -> std::array<uint8_t, 64>;
[[nodiscard]] auto hkdf_extract(salt, ikm) -> std::array<uint8_t, 32>;
[[nodiscard]] auto hkdf_expand(prk, info, length) -> std::pair<fault::code, vector<uint8_t>>;
[[nodiscard]] auto hkdf_expand_label(secret, label, context, length) -> std::pair<fault::code, vector<uint8_t>>;
[[nodiscard]] auto sha256(data) -> std::array<uint8_t, 32>;
[[nodiscard]] auto sha256(data1, data2) -> std::array<uint8_t, 32>;
```

`hkdf_expand_label` 按 RFC 8446 Section 7.1 实现，自动添加 `"tls13 "` 前缀。

### 4.5 AES-ECB 单块（`block.hpp`）

```cpp
[[nodiscard]] auto aes_ecb_encrypt(std::span<const uint8_t, 16> input, key) -> std::array<uint8_t, 16>;
[[nodiscard]] auto aes_ecb_decrypt(std::span<const uint8_t, 16> input, key) -> std::array<uint8_t, 16>;
```

### 4.6 Base64 编解码（`base64.hpp`）/ SHA-224（`sha224.hpp`）

Base64 支持 URL-safe 变体。`normalize_credential` 检测输入是否已是 56 字节十六进制字符串，是则直接返回，否则计算 SHA-224 哈希。

## 5. 文件清单

### 头文件（`include/prism/crypto/`）

```
crypto.hpp              # 聚合头文件
├── aead.hpp            # AEAD 加解密（aead_cipher, aead_context）
├── blake3.hpp          # BLAKE3 密钥派生（derive_key）
├── x25519.hpp          # X25519 密钥交换
├── hkdf.hpp            # HKDF-SHA256 密钥派生 + SHA-256
├── block.hpp           # AES-ECB 单块加解密
├── base64.hpp          # Base64 编解码（header-only）
└── sha224.hpp          # SHA-224 哈希（header-only）
```

### 源文件（`src/prism/crypto/`）

```
├── aead.cpp            # aead_context 实现（BoringSSL EVP_AEAD）
├── blake3.cpp          # derive_key 实现（BLAKE3 C API）
├── x25519.cpp          # X25519 密钥交换实现（BoringSSL EVP_PKEY）
├── hkdf.cpp            # HMAC/HKDF/SHA-256 实现（BoringSSL HMAC）
└── block.cpp           # AES-ECB 实现（BoringSSL AES）
```

## 6. 设计原理

- **前向声明隔离**：避免将 OpenSSL 头文件暴露到业务层，使用 `unique_ptr` + 自定义删除器管理 BoringSSL 对象
- **错误码而非异常**：所有加密操作返回 `fault::code`，符合热路径无异常原则
- **编译时大小计算**：`tag_length()`、`seal_output_size()`、`open_output_size()` 均为 constexpr
- **零分配接口**：tag/nonce 长度为 constexpr，输出大小编译时可计算

- [[crypto/aead]] — AEAD 加密实现详解
## 相关页面

- [[stealth]] — Stealth 模块（Reality 使用 X25519 + HKDF）
- [[pipeline]] — Pipeline 模块（SS2022 使用 AEAD + BLAKE3）
- [[fault]] — Fault 模块
- [[architecture]] — 架构设计
