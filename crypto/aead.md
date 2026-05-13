---
title: AEAD 加密实现
module: crypto::aead
namespace: psm::crypto
files:
  - include/prism/crypto/aead.hpp
  - src/prism/crypto/aead.cpp
see_also:
  - "[[crypto]]"
  - "[[stealth]]"
  - "[[channel/transport]]"
---

# AEAD 加密实现

AEAD（Authenticated Encryption with Associated Data）模块包装 BoringSSL 的
`EVP_AEAD` API，为 [[stealth]] 模块的 SS2022 协议提供类型安全的加解密接口。

## aead_cipher — 算法枚举

```cpp
enum class aead_cipher : uint8_t {
    aes_128_gcm,          // 16 字节密钥, 12 字节 nonce
    aes_256_gcm,          // 32 字节密钥, 12 字节 nonce
    chacha20_poly1305,    // 32 字节密钥, 12 字节 nonce
    xchacha20_poly1305    // 32 字节密钥, 24 字节 nonce
};
```

所有算法的认证标签（tag）长度固定为 **16 字节**。
前三者的 nonce 为 12 字节，XChaCha20 扩展到 24 字节。

## aead_context 类

`aead_context` 管理 BoringSSL `EVP_AEAD_CTX` 的生命周期和 nonce 状态。

### 生命周期

- **构造：** `aead_context(aead_cipher cipher, span<const uint8_t> key)`
  根据算法类型选择对应的 BoringSSL 实现，用密钥初始化上下文，
  nonce 初始化为零值。密钥长度必须与算法匹配（16 或 32 字节）。
- **析构：** 通过 `unique_ptr` 自动调用 `EVP_AEAD_CTX_cleanup` + `delete`
- **移动：** 支持移动构造和移动赋值，源对象置为安全状态（`nullptr` + 零 nonce）
- **拷贝：** 禁止（含原始指针，不可拷贝）

内部使用 `unique_ptr<evp_aead_ctx_st, 函数指针删除器>` 管理 BoringSSL 上下文。

### 自动 nonce 重载（TCP 流加密）

```cpp
auto seal(span<uint8_t> out, span<const uint8_t> plaintext,
          span<const uint8_t> ad = {}) -> fault::code;

auto open(span<uint8_t> out, span<const uint8_t> ciphertext,
          span<const uint8_t> ad = {}) -> fault::code;
```

- 使用内部 nonce 加密/解密
- **成功后 nonce 自动递增**（小端序）
- 输出缓冲区大小：`seal` = plaintext + 16，`open` = ciphertext - 16
- 返回 `fault::code::success` 或 `fault::code::crypto_error`

这是 SS2022 TCP 流加密的主力接口——加密端和解密端必须保持 nonce 同步递增。

### 显式 nonce 重载（UDP 逐包加密）

```cpp
auto seal(span<uint8_t> out, span<const uint8_t> plaintext,
          span<const uint8_t> nonce, span<const uint8_t> ad) -> fault::code;

auto open(span<uint8_t> out, span<const uint8_t> ciphertext,
          span<const uint8_t> nonce, span<const uint8_t> ad) -> fault::code;
```

- 使用调用方提供的显式 nonce，**不修改内部 nonce 状态**
- 适用于 UDP 逐包加解密场景——每个包携带独立 nonce，无状态依赖
- nonce 长度必须与算法匹配（12 或 24 字节）

## Nonce 管理

### 内部状态

```cpp
std::array<uint8_t, 24> nonce_{};  // 当前 nonce（最大 24 字节）
std::size_t nonce_len_{12};        // 实际 nonce 长度
```

### 递增规则

`increment_nonce()` 按 **小端序** 递增：从 `byte[0]` 开始加 1，
溢出则进位到 `byte[1]`，以此类推。这是 SS2022 (SIP022) 规范要求的 nonce 递增方式。

```
nonce[0]++ → 若 != 0 则停止，否则继续进位
```

### 查询接口

| 方法 | 说明 |
|------|------|
| `nonce_length()` | 返回当前 nonce 长度（12 或 24） |
| `nonce()` | 返回内部 nonce 数组的只读引用（调试用） |
| `tag_length()` | 返回 16（constexpr） |
| `seal_output_size(plaintext_len)` | 返回 `plaintext_len + 16` |
| `open_output_size(ciphertext_len)` | 返回 `ciphertext_len - 16` |

## 与 SS2022 的关系

SS2022（SIP022）是 Shadowsocks 的 AEAD 加密规范，要求：

1. **TCP 流：** 使用自动 nonce 重载，加密/解密端保持 nonce 计数器同步
2. **UDP 包：** 使用显式 nonce 重载，每包独立 nonce（通常为随机值或包序号）
3. **密钥派生：** 使用 BLAKE3 从主密钥派生子密钥（见 [[crypto]] 模块）
4. **nonce 递增：** 小端序，符合 BoringSSL 的 AEAD nonce 约定

`aead_context` 的设计直接对应 SS2022 的两种使用模式：
`seal/open`（自动 nonce）用于 TCP 流，带 nonce 参数的重载用于 UDP 包。

## 典型用法

### TCP 流加密

```cpp
// 初始化
aead_context enc(aead_cipher::aes_256_gcm, key);
aead_context dec(aead_cipher::aes_256_gcm, key);

// 加密（nonce 自动递增）
std::array<uint8_t, 1040> ciphertext; // 1024 + 16
enc.seal(ciphertext, plaintext);

// 解密（nonce 自动递增，必须与加密端同步）
std::array<uint8_t, 1024> decrypted;
dec.open(decrypted, ciphertext);
```

### UDP 逐包加密

```cpp
aead_context ctx(aead_cipher::xchacha20_poly1305, key);

// 每个包使用独立 nonce
std::array<uint8_t, 24> packet_nonce = generate_nonce();
ctx.seal(ciphertext, plaintext, packet_nonce, ad);
```

## 错误处理

所有 `seal`/`open` 方法返回 `fault::code`：

- `fault::code::success` — 操作成功
- `fault::code::crypto_error` — 上下文无效或 BoringSSL 操作失败（如标签不匹配）
