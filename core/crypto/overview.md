---
title: Crypto 加密模块
layer: core
module: crypto
source:
  - include/prism/crypto.hpp
  - include/prism/crypto/aead.hpp
  - include/prism/crypto/hkdf.hpp
  - include/prism/crypto/x25519.hpp
  - include/prism/crypto/blake3.hpp
  - include/prism/crypto/block.hpp
  - include/prism/crypto/base64.hpp
  - include/prism/crypto/sha224.hpp
tags: [crypto, overview]
---

# Crypto 加密模块

Prism 加密模块提供类型安全的密码学原语封装，基于 BoringSSL 实现，支持现代加密协议所需的全部核心功能。

## 模块组成

| 模块 | 功能 | 应用场景 |
|------|------|----------|
| [[core/crypto/aead|aead]] | AEAD 认证加密 | TLS 1.3、SS2022 数据加密 |
| [[core/crypto/hkdf|hkdf]] | HKDF 密钥派生 | TLS 1.3 密钥调度 |
| [[core/crypto/x25519|x25519]] | X25519 密钥交换 | Reality 协议 ECDHE |
| [[core/crypto/blake3|blake3]] | BLAKE3 密钥派生 | SS2022 会话密钥派生 |
| [[core/crypto/block|block]] | AES-ECB 单块加密 | SS2022 UDP SeparateHeader |
| [[core/crypto/base64|base64]] | Base64 编解码 | HTTP Basic 认证 |

## 设计原则

### 类型安全

所有函数使用 `std::span` 和 `std::array` 传递密钥和数据，避免裸指针和缓冲区溢出：

```cpp
// 固定长度密钥
std::array<std::uint8_t, 32> key;
auto ctx = aead_context(aead_cipher::aes_256_gcm, key);

// 动态长度数据
std::vector<std::uint8_t> plaintext;
std::vector<std::uint8_t> ciphertext(plaintext.size() + aead_context::tag_length());
ctx.seal(ciphertext, plaintext);
```

### 错误处理

加密操作返回 `fault::code` 枚举，区分成功与各类错误：

```cpp
auto result = ctx.seal(out, plaintext, ad);
if (result != fault::code::success) {
    // 处理加密错误
}
```

### 资源管理

使用 RAII 管理加密上下文生命周期，支持移动语义：

```cpp
aead_context ctx1(aead_cipher::aes_256_gcm, key);
aead_context ctx2 = std::move(ctx1);  // 移动构造
// ctx1 置为无效状态，ctx2 接管资源
```

## 加密算法支持

### AEAD 算法

| 算法 | 密钥长度 | Nonce 长度 | Tag 长度 | 用途 |
|------|----------|------------|----------|------|
| AES-128-GCM | 16 字节 | 12 字节 | 16 字节 | TLS 1.3 |
| AES-256-GCM | 32 字节 | 12 字节 | 16 字节 | TLS 1.3、SS2022 |
| ChaCha20-Poly1305 | 32 字节 | 12 字节 | 16 字节 | TLS 1.3、SS2022 |
| XChaCha20-Poly1305 | 32 字节 | 24 字节 | 16 字节 | SS2022 |

### 密钥派生

| 算法 | 输出长度 | 用途 |
|------|----------|------|
| HKDF-SHA256 | 可变（最大 8160 字节） | TLS 1.3 密钥调度 |
| BLAKE3 derive_key | 可变 | SS2022 会话子密钥 |

### 密钥交换

| 算法 | 密钥长度 | 共享密钥长度 | 安全强度 |
|------|----------|--------------|----------|
| X25519 | 32 字节 | 32 字节 | 128 位 |

## 设计决策

### 为什么基于 BoringSSL 而非 OpenSSL？

Prism 在 TLS 层使用 BoringSSL（BoringSSL 提供 OpenSSL API 兼容层），crypto 模块直接复用同一底层库的 EVP API，避免引入第二个密码学依赖。

**后果**: 所有 EVP 函数签名与 OpenSSL 一致，但 BoringSSL 不保证跨版本 ABI 稳定，升级时需全量回归测试。

### 为什么用 RAII 封装 EVP 而非裸 API？

BoringSSL 的 `EVP_AEAD_CTX`、`EVP_CIPHER_CTX` 等是 C 结构体，需手动 `init`/`cleanup`。`aead_context` 用 `unique_ptr` + 函数指针删除器管理，`block` 模块的 `ecb_encrypt`/`ecb_decrypt` 在单次调用内创建并释放 `EVP_CIPHER_CTX`，确保无资源泄露路径。

**后果**: 调用方无需关心 BoringSSL 内部生命周期，但 `aead_context` 不可拷贝（包含 `unique_ptr`），需通过移动语义传递。

### 为什么 AEAD 和密钥派生是两套独立 API（HKDF vs BLAKE3）？

TLS 1.3 密钥调度标准规定使用 HKDF-SHA256（RFC 8446），而 SS2022 (SIP022) 规定使用 BLAKE3 derive_key。两者用途不同、调用方不同，强行统一反而增加复杂度。

**后果**: `hkdf.hpp` 仅被 Reality/stealth 模块调用，`blake3.hpp` 仅被 SS2022 和 Restls 调用，互不耦合。

### 为什么 header-only 与编译单元混排？

`base64`、`sha224`、`block` 是简单无状态函数，适合 header-only inline；`aead`、`hkdf`、`x25519`、`blake3` 依赖 BoringSSL/blake3 C API 且内部有复杂状态，必须编译为独立 `.cpp`，避免模板实例化膨胀和 ODR 问题。

**后果**: 包含 `aead.hpp` 不暴露 BoringSSL 头文件（前向声明 `evp_aead_ctx_st`），减少编译依赖传播。

## 依赖关系

### 约束

### 前向声明隔离

**类型**: 调用顺序

**规则**: `aead.hpp` 通过前向声明 `struct evp_aead_ctx_st` 隐藏 BoringSSL 实现细节，包含 `aead.hpp` 不暴露 `<openssl/evp.h>`

**违反后果**: 如果内部实现改为直接包含 BoringSSL 头文件，所有包含 `aead.hpp` 的编译单元的编译时间显著增加

**源码依据**: `aead.hpp:21`

### 构造时初始化顺序

**类型**: 调用顺序

**规则**: crypto 模块不依赖全局状态，但依赖 BoringSSL 在进程启动时已完成初始化

**违反后果**: 在 BoringSSL 初始化前调用任何 crypto 函数会导致未定义行为

**源码依据**: `main.cpp` 启动流程中 BoringSSL 初始化先于 crypto 使用

### 故障场景

### 全局密码学失败

**触发条件**: BoringSSL 初始化失败（FIPS 模式自检失败、rng 初始化失败）

**传播路径**: 所有 crypto 函数返回 `crypto_error` 或全零结果 -> 全部协议处理失败 -> 无法接受任何连接

**外部表现**: Prism 进程启动后所有连接立即断开

**恢复机制**: 重启进程，检查 BoringSSL 版本和 FIPS 配置

**日志关键字**: `"EVP_AEAD_CTX_init 失败"`、`"HMAC-SHA256 计算失败"`、`"X25519 密钥交换失败"`

```
┌─────────────────────────────────────────────────────────┐
│                     Protocol Layer                       │
│  TLS 1.3 │ SS2022 │ Reality │ Trojan │ VLESS │ SOCKS5  │
└────┬──────────┬─────────┬──────────┬──────────────┬─────┘
     │          │         │          │              │
     ▼          ▼         ▼          ▼              ▼
┌─────────────────────────────────────────────────────────┐
│                     Crypto Layer                         │
│  ┌──────┐ ┌──────┐ ┌────────┐ ┌────────┐ ┌───────────┐  │
│  │ AEAD │ │ HKDF │ │ X25519 │ │ BLAKE3 │ │ AES Block │  │
│  └──┬───┘ └──┬───┘ └───┬────┘ └───┬────┘ └─────┬─────┘  │
│     │        │         │          │            │        │
│     ▼        ▼         ▼          ▼            ▼        │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   BoringSSL / BLAKE3                     │
└─────────────────────────────────────────────────────────┘
```

### 跨模块契约

| 模块 A | 模块 B | 契约内容 |
|--------|--------|---------|
| [[core/protocol/shadowsocks/format\|SS2022]] | [[core/crypto/blake3\|blake3]] | SS2022 必须使用 SIP022 规定的固定 context 字符串 `"shadowsocks 2022 session subkey"` 调用 `derive_key` |
| [[core/stealth/reality/handshake\|Reality]] | [[core/crypto/x25519\|x25519]] | Reality 握手调用 `x25519()` 后必须检查返回 `fault::code`，全零共享密钥视为低阶点攻击 |
| [[core/stealth/reality/handshake\|Reality]] | [[core/crypto/hkdf\|hkdf]] | Reality 通过 `expand_label` 派生 AEAD 密钥/IV，label 必须以 RFC 8446 规定的 `"tls13 "` 为前缀 |
| [[core/stealth/reality/handshake\|Reality]] | [[core/crypto/aead\|aead]] | Reality 使用 AES-128-GCM seal 响应体，密钥由 HKDF ExpandLabel 派生 |
| [[core/stealth/restls/crypto\|Restls]] | [[core/crypto/blake3\|blake3]] | Restls 使用 `keyed_hasher` 构造 MAC，context 为 `"restls-traffic-key"` |
| [[core/loader/load\|loader]] | [[core/crypto/sha224\|sha224]] | 配置加载时调用 `normalize_credential` 统一 Trojan 密码格式 |
| [[core/protocol/http/parser\|HTTP]] | [[core/crypto/base64\|base64]] | HTTP Basic Auth 解码使用 `base64_decode`，支持标准与 URL-safe 变体 |
| [[core/protocol/socks5/conn\|SOCKS5]] | [[core/crypto/base64\|base64]] | SOCKS5 用户名/密码认证中使用 Base64 |
| [[core/protocol/shadowsocks/datagram\|SS2022 UDP]] | [[core/crypto/block\|block]] | SS2022 UDP SeparateHeader 使用 `ecb_encrypt`/`ecb_decrypt` 处理固定 16 字节头部 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| 新增 AEAD 算法枚举值 | `aead_cipher` 枚举 | 所有 `switch(cipher)` 的调用方需补充分支 |
| 修改 `increment_nonce` 逻辑 | `aead_context` | SS2022 TCP 流量加解密两端 nonce 失同步，连接断开 |
| 修改 BLAKE3 context 字符串 | `blake3::derive_key` | SS2022 密钥派生结果不同，与标准实现不兼容 |
| 修改 `fault::code` 枚举值 | 全部 crypto 函数 | 所有检查返回值的调用方编译失败 |
| `normalize_credential` 逻辑变更 | `sha224` | Trojan 用户无法认证，配置兼容性断裂 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| BoringSSL 升级 | EVP API 签名变化 | 全部 `EVP_*` 调用点（aead/hkdf/block） |
| BLAKE3 v1.8.1 → 新版本 | C API 兼容性 | `blake3_hasher_*` 系列函数签名 |
| SS2022 SIP022 规范修订 | context 字符串、KDF 流程 | `constants.hpp` 中的固定字符串和 `conn.cpp` |
| TLS 1.3 RFC 修订 | HKDF label 定义 | `expand_label` 的 `"tls13 "` 前缀拼接逻辑 |

## 相关文档

- [[core/crypto/aead|aead]] - AEAD 认证加密详解
- [[core/crypto/hkdf|hkdf]] - HKDF 密钥派生详解
- [[core/crypto/x25519|x25519]] - X25519 密钥交换详解
- [[core/crypto/blake3|blake3]] - BLAKE3 密钥派生详解
- [[core/crypto/block|block]] - AES-ECB 单块加密详解
- [[core/crypto/base64|base64]] - Base64 编解码详解