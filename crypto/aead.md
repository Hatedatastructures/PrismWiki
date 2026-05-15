---
title: "aead — AEAD 加密解密工具"
source: "include/prism/crypto/aead.hpp"
module: "crypto"
type: api
tags: [crypto, aead, 加密, 认证加密]
related:
  - crypto/blake3
  - crypto/block
  - protocol/shadowsocks
created: 2026-05-15
updated: 2026-05-15
---

# aead.hpp

> 源码: `include/prism/crypto/aead.hpp`
> 实现: `src/prism/crypto/aead.cpp`
> 模块: [[crypto|crypto]]

## 概述

AEAD 加密解密工具。包装 BoringSSL 的 EVP_AEAD API，提供类型安全的加解密接口。支持 AES-128-GCM、AES-256-GCM、ChaCha20-Poly1305、XChaCha20-Poly1305。管理 nonce 状态，每次 seal/open 后自动递增 nonce（小端序）。同时提供显式 nonce 重载，用于无状态的逐包加解密（如 SS2022 UDP）。

## 命名空间

`psm::crypto`

## 依赖关系

| 依赖 | 用途 | 层级 |
|------|------|------|
| `<prism/fault/code.hpp>` | 错误码 `fault::code` | fault |
| `<openssl/evp.h>` | BoringSSL EVP_AEAD API | 外部 |
| `<prism/trace/spdlog.hpp>` | 错误日志 | trace |

## 枚举: aead_cipher

```cpp
enum class aead_cipher : std::uint8_t {
    aes_128_gcm,           // AES-128-GCM
    aes_256_gcm,           // AES-256-GCM
    chacha20_poly1305,     // ChaCha20-Poly1305
    xchacha20_poly1305     // XChaCha20-Poly1305
};
```

| 值 | 说明 | 密钥长度 | nonce 长度 |
|-----|------|----------|-----------|
| `aes_128_gcm` | AES-128-GCM | 16 字节 | 12 字节 |
| `aes_256_gcm` | AES-256-GCM | 32 字节 | 12 字节 |
| `chacha20_poly1305` | ChaCha20-Poly1305 | 32 字节 | 12 字节 |
| `xchacha20_poly1305` | XChaCha20-Poly1305 | 32 字节 | 24 字节 |

## 类: aead_context

AEAD 加密上下文。管理 BoringSSL EVP_AEAD_CTX 的生命周期和 nonce 状态。不可拷贝，可移动。

### 构造函数

```cpp
explicit aead_context(aead_cipher cipher, std::span<const std::uint8_t> key);
```

- **cipher**: 加密算法类型
- **key**: 密钥（长度必须与算法匹配：AES-128 为 16 字节，其余为 32 字节）
- **行为**: 根据 cipher 类型选择 BoringSSL AEAD 实现，初始化 EVP_AEAD_CTX，nonce 初始化为全零

### 析构函数

```cpp
~aead_context();
```

- **行为**: 通过 `unique_ptr` 自动释放 BoringSSL EVP_AEAD_CTX

### 特殊成员函数

```cpp
aead_context(const aead_context &) = delete;                             // 禁止拷贝构造
auto operator=(const aead_context &) -> aead_context & = delete;         // 禁止拷贝赋值
aead_context(aead_context &&other) noexcept;                             // 移动构造
auto operator=(aead_context &&other) noexcept -> aead_context &;         // 移动赋值
```

- 移动操作转移 BoringSSL 上下文和 nonce 状态的所有权，源对象置为安全状态（ctx_ = nullptr, nonce 全零）

### seal() — 自动 nonce 加密

```cpp
auto seal(std::span<std::uint8_t> out,
          std::span<const std::uint8_t> plaintext,
          std::span<const std::uint8_t> ad = {})
    -> fault::code;
```

| 参数 | 说明 |
|------|------|
| `out` | 输出缓冲区（密文 + tag），大小必须 >= `plaintext.size() + tag_length()` |
| `plaintext` | 明文数据 |
| `ad` | 附加认证数据（可选，默认为空） |

- **返回值**: 成功返回 `fault::code::success`，失败返回 `fault::code::crypto_error`
- **副作用**: 成功后自动递增内部 nonce（小端序）
- **调用（向下）**: `EVP_AEAD_CTX_seal()` (BoringSSL) → `increment_nonce()`
- **被调用（向上）**: `protocol::shadowsocks::relay`、`stealth::reality::seal`
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]

### open() — 自动 nonce 解密

```cpp
auto open(std::span<std::uint8_t> out,
          std::span<const std::uint8_t> ciphertext,
          std::span<const std::uint8_t> ad = {})
    -> fault::code;
```

| 参数 | 说明 |
|------|------|
| `out` | 输出缓冲区（明文），大小必须 >= `ciphertext.size() - tag_length()` |
| `ciphertext` | 密文 + tag |
| `ad` | 附加认证数据（可选，默认为空） |

- **返回值**: 成功返回 `fault::code::success`，失败返回 `fault::code::crypto_error`（密文被篡改或 tag 不匹配时也会失败）
- **副作用**: 成功后自动递增内部 nonce（小端序）
- **调用（向下）**: `EVP_AEAD_CTX_open()` (BoringSSL) → `increment_nonce()`
- **被调用（向上）**: `protocol::shadowsocks::relay`
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]

### seal() — 显式 nonce 加密

```cpp
auto seal(std::span<std::uint8_t> out,
          std::span<const std::uint8_t> plaintext,
          std::span<const std::uint8_t> nonce,
          std::span<const std::uint8_t> ad)
    -> fault::code;
```

| 参数 | 说明 |
|------|------|
| `out` | 输出缓冲区（密文 + tag） |
| `plaintext` | 明文数据 |
| `nonce` | 显式 nonce（12 或 24 字节，取决于算法） |
| `ad` | 附加认证数据 |

- **返回值**: 成功返回 `fault::code::success`，失败返回 `fault::code::crypto_error`
- **副作用**: 无（不修改内部 nonce 状态）
- **适用场景**: UDP 逐包加密等无状态场景
- **调用（向下）**: `EVP_AEAD_CTX_seal()` (BoringSSL)
- **被调用（向上）**: `protocol::shadowsocks::relay`、`stealth::reality::seal`
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]

### open() — 显式 nonce 解密

```cpp
auto open(std::span<std::uint8_t> out,
          std::span<const std::uint8_t> ciphertext,
          std::span<const std::uint8_t> nonce,
          std::span<const std::uint8_t> ad)
    -> fault::code;
```

| 参数 | 说明 |
|------|------|
| `out` | 输出缓冲区（明文） |
| `ciphertext` | 密文 + tag |
| `nonce` | 显式 nonce（12 或 24 字节） |
| `ad` | 附加认证数据 |

- **返回值**: 成功返回 `fault::code::success`，失败返回 `fault::code::crypto_error`
- **副作用**: 无（不修改内部 nonce 状态）
- **适用场景**: UDP 逐包解密等无状态场景
- **调用（向下）**: `EVP_AEAD_CTX_open()` (BoringSSL)
- **被调用（向上）**: `protocol::shadowsocks::relay`
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]

### tag_length() — 返回认证标签长度

返回所有支持的 AEAD 算法统一使用的认证标签长度（固定 16 字节）。

```cpp
[[nodiscard]] static constexpr auto tag_length() noexcept -> std::size_t;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| （无） | — | — |

- **返回值**: `std::size_t`，始终返回 16。AES-GCM 和 ChaCha20-Poly1305 均使用 128 位（16 字节）认证标签。
- **调用（向下）**: 无（编译期常量）
- **被调用（向上）**: `seal_output_size()`、`open_output_size()`、`seal()` / `open()` 文档中引用用于说明输出缓冲区大小计算
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]

### nonce_length() — 返回当前算法的 nonce 长度

返回当前上下文所使用的 AEAD 算法对应的 nonce 字节长度。

```cpp
[[nodiscard]] auto nonce_length() const noexcept -> std::size_t;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| （无） | — | — |

- **返回值**: `std::size_t`，GCM/ChaCha20-Poly1305 返回 12，XChaCha20-Poly1305 返回 24。该值在构造时由 `aead_cipher` 类型决定。
- **调用（向下）**: 无（读取成员 `nonce_len_`）
- **被调用（向上）**: 自动 nonce 版 `seal()` / `open()` 内部使用，将 `nonce_len_` 传递给 BoringSSL EVP 接口；测试 `Aead.cpp` 用于验证算法参数
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]、[[ref/anti-censorship/replay-attack|重放攻击]]

### nonce() — 返回当前 nonce 值的只读引用

返回内部 nonce 数组的常量引用，用于调试和状态检查。

```cpp
[[nodiscard]] auto nonce() const noexcept -> const std::array<std::uint8_t, 24> &;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| （无） | — | — |

- **返回值**: `const std::array<std::uint8_t, 24> &`，内部 nonce 数组的只读引用。数组固定为 24 字节，实际有效长度由 `nonce_length()` 决定。
- **调用（向下）**: 无（读取成员 `nonce_`）
- **被调用（向上）**: 测试 `Aead.cpp` 中 `TestNonceIncrement` 验证 seal 后 nonce 递增行为；外部调试代码可读取当前 nonce 状态
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]、[[ref/anti-censorship/replay-attack|重放攻击]]

### seal_output_size() — 计算加密输出缓冲区大小

根据明文长度计算 seal 操作所需的输出缓冲区大小（明文 + 认证标签）。

```cpp
[[nodiscard]] static constexpr auto seal_output_size(std::size_t plaintext_len) noexcept -> std::size_t;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `plaintext_len` | `std::size_t` | 明文字节长度 |

- **返回值**: `std::size_t`，`plaintext_len + tag_length()`，即明文长度加 16 字节标签。
- **调用（向下）**: `tag_length()`（获取标签长度常量 16）
- **被调用（向上）**: `shadowsocks::datagram` 计算 UDP 加密缓冲区大小、`shadowsocks::relay` 计算 TCP 流加密缓冲区大小、测试 `Aead.cpp` 分配输出缓冲区
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/network/udp|UDP]]、[[ref/anti-censorship/replay-attack|重放攻击]]

### open_output_size() — 计算解密输出缓冲区大小

根据密文长度（含标签）计算 open 操作所需的输出缓冲区大小（密文 - 认证标签）。

```cpp
[[nodiscard]] static constexpr auto open_output_size(std::size_t ciphertext_len) noexcept -> std::size_t;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `ciphertext_len` | `std::size_t` | 密文字节长度（含 16 字节认证标签） |

- **返回值**: `std::size_t`，`ciphertext_len - tag_length()`，即密文长度减 16 字节标签。
- **调用（向下）**: `tag_length()`（获取标签长度常量 16）
- **被调用（向上）**: `stealth::reality::handshake` 计算解密后的明文缓冲区大小、测试 `Aead.cpp` 分配解密输出缓冲区
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/network/udp|UDP]]、[[ref/anti-censorship/replay-attack|重放攻击]]

### increment_nonce() — 小端序递增 nonce（私有）

按小端序逐字节递增内部 nonce 值，溢出时向高位进位。由自动 nonce 版 seal/open 成功后调用。

```cpp
void increment_nonce();
```

| 参数 | 类型 | 说明 |
|------|------|------|
| （无） | — | — |

- **返回值**: 无（void）
- **调用（向下）**: 无（纯数组操作，逐字节 +1 并检测溢出进位）
- **被调用（向上）**: 自动 nonce 版 `seal()` 和 `open()` 在加密/解密成功后调用
- **涉及的知识域**: [[ref/crypto/aes-gcm|AES-GCM]]、[[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]]、[[ref/anti-censorship/replay-attack|重放攻击]]

## 调用链

```
向上调用者:
  shadowsocks::session::decrypt_packet()  → aead_context::open()
  shadowsocks::session::encrypt_packet()  → aead_context::seal()
  SS2022 UDP 逐包加解密                   → aead_context::seal/open(nonce 重载)

向下依赖:
  aead_context::seal/open()  → EVP_AEAD_CTX_seal/open()  (BoringSSL)
  aead_context 构造          → EVP_aead_aes_128_gcm() 等 (BoringSSL)
  aead_context::increment_nonce()  → 小端序 nonce 递增
```

## 私有成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `ctx_` | `unique_ptr<evp_aead_ctx_st, ...>` | BoringSSL AEAD 上下文（带函数指针删除器） |
| `nonce_` | `std::array<std::uint8_t, 24>` | 当前 nonce 值（最大 24 字节） |
| `key_length_` | `std::size_t` | 密钥长度 |
| `nonce_len_` | `std::size_t` | nonce 长度（12 或 24） |

## 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-GCM 认证加密模式
- [[ref/crypto/chacha20-poly1305|ChaCha20-Poly1305]] — ChaCha20-Poly1305 认证加密
- [[ref/anti-censorship/replay-attack|重放攻击]] — nonce 管理是防御重放攻击的核心机制
- [[ref/network/udp|UDP]] — 显式 nonce 重载面向 UDP 逐包加密场景
