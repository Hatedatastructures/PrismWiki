---
layer: core
source: include/prism/crypto/block.hpp
  - include/prism/crypto/block.hpp
  - src/prism/crypto/block.cpp
tags: [crypto, block, aes-ecb]
title: AES-ECB 单块加密
---

# AES-ECB 单块加密

AES-ECB 单块加密提供对单个 16 字节数据块的 AES 加密和解密。本模块专用于 SS2022 (SIP022) UDP 的 SeparateHeader 加密，**不应用于大量数据加密**（ECB 模式不安全）。

## 设计决策

### 为什么只提供 ECB 而非 CBC/GCM 等模式？

SS2022 UDP 协议的 SeparateHeader 恰好是 16 字节（1 字节 Type + 1 字节 Padding + 14 字节 Session ID），等于 AES 的一个块大小。ECB 对单块加密等价于原始 AES，无模式开销。CBC/GCM 需要 IV/nonce，对固定 16 字节输入是多余复杂度。

**后果**: 此模块严禁用于超过 16 字节的数据。多于一块时 ECB 的确定性加密会泄露模式。

### 为什么用 `span<const uint8_t, 16>` 而非 `array` 参数？

`std::span<const uint8_t, 16>` 的静态 extent 在编译期保证输入恰好 16 字节，同时接受任何连续内存（`std::array`、C 数组、`vector` 前缀）。比 `std::array` 参数更灵活，比无 extent 的 `span` 更安全。

**后果**: 编译期类型检查阻止传入错误大小的数据。

### 为什么每次调用创建并释放 EVP_CIPHER_CTX？

AES-ECB 单块加密是 UDP 逐包操作，每个数据包调用一次。`EVP_CIPHER_CTX` 的创建/释放开销（约 200ns）相比网络 I/O 延迟可忽略。避免持久化上下文带来的线程安全和状态管理复杂度。

**后果**: 无状态函数天然线程安全，可并发调用。

## 约束

### 输入固定 16 字节

**类型**: 状态前置

**规则**: `input` 参数必须是恰好 16 字节（`span<const uint8_t, 16>` 编译期保证）

**违反后果**: 编译失败

**源码依据**: `block.hpp:27`（`std::span<const std::uint8_t, 16>` 类型）

### 密钥长度限制

**类型**: 调用顺序

**规则**: `key` 参数必须为 16 字节（AES-128）或 32 字节（AES-256），其他长度走 AES-256 路径

**违反后果**: 非 16/32 字节密钥时 `EVP_aes_*_ecb()` 选择可能不正确，BoringSSL 行为未定义

**源码依据**: `block.cpp:16-23`（if/else 分支）

## 源码位置

- 头文件：`include/prism/crypto/block.hpp`

## 函数详解

### aes_ecb_encrypt

```cpp
[[nodiscard]] auto aes_ecb_encrypt(std::span<const std::uint8_t, 16> input,
                                    std::span<const std::uint8_t> key)
    -> std::array<std::uint8_t, 16>;
```

对单个 16 字节块执行 AES-ECB 加密。

**参数**：
- `input`：明文（固定 16 字节）
- `key`：AES 密钥（16 或 32 字节）

**返回值**：密文（16 字节）

**支持的密钥长度**：
| 密钥长度 | 算法 |
|----------|------|
| 16 字节 | AES-128 |
| 32 字节 | AES-256 |

### aes_ecb_decrypt

```cpp
[[nodiscard]] auto aes_ecb_decrypt(std::span<const std::uint8_t, 16> input,
                                    std::span<const std::uint8_t> key)
    -> std::array<std::uint8_t, 16>;
```

对单个 16 字节块执行 AES-ECB 解密。

**参数**：
- `input`：密文（固定 16 字节）
- `key`：AES 密钥（16 或 32 字节）

**返回值**：明文（16 字节）

## 安全警告

### ECB 模式的弱点

**ECB（Electronic Codebook）模式不适用于加密超过一个块的数据**：

```
明文: [Block A][Block A][Block B][Block B]
ECB:  [Enc A  ][Enc A  ][Enc B  ][Enc B  ]
                    ^^^^^^         ^^^^^^
                    相同的明文块产生相同的密文块
```

这导致：
1. **模式泄露**：相同的明文块产生相同的密文块，泄露数据模式
2. **无完整性保护**：攻击者可以重新排列密文块
3. **选择明文攻击**：攻击者可以推断明文内容

### 正确用法

本模块仅用于：
- **SS2022 UDP SeparateHeader 加密**：加密固定 16 字节的头部信息
- **其他固定长度、单块加密场景**

```cpp
// 正确：加密固定 16 字节头部
std::array<std::uint8_t, 16> header = /* ... */;
std::array<std::uint8_t, 16> encrypted_header = aes_ecb_encrypt(header, key);

// 错误：加密多块数据
std::vector<std::uint8_t> data = /* 多于 16 字节 */;
// 不要使用 ECB 模式！应使用 AEAD 模式
```

## SS2022 UDP SeparateHeader

在 SS2022 UDP 协议中，SeparateHeader 使用 AES-ECB 加密头部：

```
┌─────────────────────────────────────┐
│         SeparateHeader (16B)        │
├──────────┬──────────┬───────────────┤
│ Type (1B)│ Pad (1B) │ Session ID (14B)│
└──────────┴──────────┴───────────────┘
              │
              ▼
      AES-ECB-Encrypt(key)
              │
              ▼
┌─────────────────────────────────────┐
│      Encrypted Header (16B)          │
└─────────────────────────────────────┘
```

## 使用示例

### 加密 16 字节块

```cpp
// 准备密钥和明文
std::array<std::uint8_t, 32> key = /* AES-256 密钥 */;
std::array<std::uint8_t, 16> plaintext = /* 16 字节明文 */;

// 加密
auto ciphertext = aes_ecb_encrypt(plaintext, key);

// 解密
auto decrypted = aes_ecb_decrypt(ciphertext, key);

// 验证
assert(plaintext == decrypted);
```

### SS2022 UDP 头部加密

```cpp
// 构造 SeparateHeader
std::array<std::uint8_t, 16> header{};
header[0] = 0x00;  // Type
header[1] = 0x00;  // Padding
// header[2..15] = Session ID

// 加密头部
auto encrypted_header = aes_ecb_encrypt(header, key);

// 发送：[encrypted_header][encrypted_payload]

// 接收端解密头部
auto decrypted_header = aes_ecb_decrypt(encrypted_header, key);
```

## 与 AEAD 比较

| 特性 | AES-ECB 单块 | AEAD (GCM/ChaCha) |
|------|--------------|-------------------|
| 认证 | 无 | 有 |
| 完整性 | 无 | 有 |
| 重放保护 | 无 | 有（通过 nonce） |
| 数据长度 | 固定 16 字节 | 任意长度 |
| 用途 | 特殊场景 | 通用加密 |

## 实现细节

### AES-NI 硬件加速

现代 CPU 提供 AES-NI 指令集，大幅提升 AES 性能：

```
普通软件实现：~100 MB/s
AES-NI 硬件加速：~3 GB/s
```

BoringSSL/OpenSSL 会自动检测并使用 AES-NI 指令。

### 密钥调度

```
AES-128: 10 轮加密
AES-256: 14 轮加密
```

密钥调度在上下文初始化时完成，加密/解密时直接使用预计算的轮密钥。

## 调用链

```mermaid
graph TD
    A[ecb_encrypt/decrypt] --> B[EVP_EncryptInit_ex / DecryptInit_ex]
    B --> C[EVP_EncryptUpdate / DecryptUpdate]
    C --> D[EVP_EncryptFinal_ex / DecryptFinal_ex]
    D --> E[BoringSSL AES]
    E --> F{AES-NI 可用?}
    F -->|是| G[硬件 AES-NI]
    F -->|否| H[软件 AES]
```

## 故障场景

### EVP 上下文创建失败

**触发条件**: `EVP_CIPHER_CTX_new()` 返回 nullptr（内存不足）

**传播路径**: 返回全零的 `std::array<uint8_t, 16>` -> 调用方收到全零密文/明文

**外部表现**: SS2022 UDP 头部解密失败，数据包被丢弃

**恢复机制**: 内存压力释放后恢复正常

**日志关键字**: 无直接日志（返回全零静默失败）

### 密钥长度不匹配

**触发条件**: 传入非 16/32 字节密钥

**传播路径**: `EVP_aes_256_ecb()` 被默认选择 -> `EVP_EncryptInit_ex` 可能失败

**外部表现**: SS2022 UDP 数据包头部加密/解密失败

**恢复机制**: 检查配置中密钥格式

**日志关键字**: 无直接日志

### 跨模块契约

| 模块 A | 模块 B | 契约内容 |
|--------|--------|---------|
| [[core/protocol/shadowsocks/datagram\|SS2022 UDP]] | [[core/crypto/block\|block]] | SS2022 UDP 使用 `ecb_decrypt` 解密接收到的 16 字节 SeparateHeader，使用 `ecb_encrypt` 加密发送的 SeparateHeader |
| [[core/crypto/blake3\|blake3]] | [[core/crypto/block\|block]] | block 模块使用的 AES 密钥由 SS2022 配置直接提供（PSK 的前 16/32 字节），不经过 blake3 派生 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| 修改 `span<const uint8_t, 16>` 的 extent | 全部调用方 | 编译失败 |
| 修改密钥长度选择逻辑（if/else） | SS2022 UDP | 密钥-算法不匹配，加解密结果错误 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| BoringSSL 升级 | `EVP_aes_*_ecb()` API | `block.cpp:17-23` |
| SS2022 SIP022 修订 | SeparateHeader 格式 | 头部是否仍为 16 字节 |

## 相关文档

- [[core/crypto/aead|aead]] - AEAD 认证加密（推荐用于通用加密）
- [[core/crypto/blake3|blake3]] - BLAKE3 密钥派生