---
layer: core
source: include/prism/stealth/common.hpp
title: Stealth 公共组件
module: stealth
tags:
  - stealth
  - common
  - AEAD
  - TLS-record
  - XOR
  - utility
created: 2026-05-23
updated: 2026-05-23
---

# Stealth 公共组件

> 源码位置: `include/prism/stealth/common.hpp`

## 概述

`stealth::common` 命名空间提供伪装层的共享基础设施函数，包括 AEAD nonce 构造、TLS 记录附加数据生成、循环异或运算和原始 TLS 帧读取。这些工具被 [[core/stealth/overview|所有伪装方案]]（Reality、ShadowTLS、Restls 等）复用。

## 设计决策（WHY）

### 为什么 `make_aead_nonce` 将 sequence 放在低 8 字节

TLS 1.3 规范（RFC 8446 Section 5.3）定义 nonce 构造为 `IV XOR (sequence as left-padded 12-byte)`。但实际实现中，64 位序列号放在低 8 字节（小端序）与 IV 的低 8 字节异或。这个选择是因为 TLS 记录序列号是 64 位整数，自然增长在低字节变化最频繁，与 IV 异或后能最大化 nonce 的变化位数。高 4 字节固定为零意味着 IV 的高 4 字节不会被序列号修改。

### 为什么 `make_record_ad` 硬编码 `0x17` 和 `0x0303`

- `0x17` (Application Data) 是 TLS 1.3 加密记录的唯一内容类型。握手完成后所有记录都是此类型。
- `0x0303` 是 TLS 1.2 的记录层版本号。TLS 1.3 为兼容中间件，在记录层使用 TLS 1.2 版本号（真实版本在 handshake 中协商）。这两个值在 TLS 1.3 生命周期内不会改变，硬编码合理。

### 为什么 `read_raw_tls_frame` 返回 `optional` 而非抛异常

这是热路径函数，在每次握手过程中可能被调用多次（Reality 5 阶段握手每阶段至少一次）。使用 `error_code` + `optional` 的双轨错误策略，避免异常的栈展开开销。调用方检查 `optional` 即可，无需 try-catch。

### 为什么 `xor_with_key` 就地修改而非返回新 buffer

避免额外的内存分配。在 ShadowTLS 握手的 relay 阶段，每帧数据都要 XOR，如果每帧都分配新 buffer 会在热路径产生大量 PMR 分配。就地修改用 `span` 零拷贝。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| `iv` 必须 12 字节 | TLS 1.3 AEAD 规范 | AES-128-GCM 和 AES-256-GCM 的 nonce 都是 12 字节 |
| `sequence` 同方向单调递增 | TLS 1.3 规范 | 读写方向各自独立计数，不可重用 |
| `xor_with_key` key 不可为空 | 除零风险 | `i % key.len()` 除零 |
| `read_raw_tls_frame` 仅适用于 TCP | 参数类型为 `tcp::socket&` | QUIC/UDP 帧读取不适用此函数 |
| `encrypted_len` 不含 AEAD tag | `make_record_ad` 语义 | 调用方需自行加上 tag 大小（16 字节） |
| 所有函数在 `stealth::common` 命名空间 | 模块组织 | 非 `stealth_scheme` 的成员函数 |

## 失败场景

| 场景 | 触发条件 | 表现 |
|------|----------|------|
| `sequence` 溢出 | 超过 2^64 | nonce 重复，AEAD 安全性被破坏 |
| `iv` 不足 12 字节 | 调用方传错 | 内存越界读取（undefined behavior） |
| `read_raw_tls_frame` 读取超时 | 网络中断 | `ec` 被设置，返回 `nullopt` |
| `read_raw_tls_frame` 读到截断帧 | 连接在读取载荷中关闭 | `ec` 被设置为 `eof`，返回 `nullopt` |
| `xor_with_key` key 与 data 长度关系 | key 长度整除 data 长度时模式更明显 | 弱密钥可被统计分析 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `common` → `crypto::aead` | 被调用 | `make_aead_nonce` 和 `make_record_ad` 的输出传给 AEAD 的 encrypt/decrypt |
| `reality::handshake` → `common` | 调用 | Reality seal/unseal 使用全部四个函数 |
| `shadowtls::handshake` → `common` | 调用 | ShadowTLS relay 使用 `xor_with_key` 和 `read_raw_tls_frame` |
| `restls::handshake` → `common` | 调用 | Restls 使用 `read_raw_tls_frame` |
| `common` → `memory::container` | 依赖 | `read_raw_tls_frame` 返回 `memory::vector`（PMR 分配） |

## 变更敏感性

| 变更 | 影响范围 | 风险 |
|------|----------|------|
| 修改 `make_aead_nonce` 的 XOR 逻辑 | 所有使用 AEAD 的方案 | 极高：nonce 错误导致解密必然失败 |
| 修改 `make_record_ad` 的固定字节 | 所有 AEAD 加密的方案 | 极高：AAD 错误导致 AEAD 认证失败 |
| 修改 `read_raw_tls_frame` 的返回类型 | Reality、ShadowTLS、Restls | 高：调用方解包逻辑需同步修改 |
| 修改 `xor_with_key` 为分配版本 | ShadowTLS relay 热路径 | 中：性能退化 |

## 命名空间

```cpp
namespace psm::stealth::common
{
    // AEAD nonce 构造
    [[nodiscard]] auto make_aead_nonce(
        std::span<const std::uint8_t> iv,
        std::uint64_t sequence) noexcept
        -> std::array<std::uint8_t, 12>;

    // TLS 记录附加数据 (AEAD AAD)
    [[nodiscard]] auto make_record_ad(
        std::uint16_t encrypted_len) noexcept
        -> std::array<std::uint8_t, 5>;

    // 循环异或
    void xor_with_key(
        std::span<std::byte> data,
        std::span<const std::uint8_t> key) noexcept;

    // 读取原始 TLS 帧
    auto read_raw_tls_frame(
        net::ip::tcp::socket &sock,
        std::error_code &ec)
        -> net::awaitable<std::optional<memory::vector<std::byte>>>;
}
```

## 函数详解

### make_aead_nonce

```cpp
[[nodiscard]] inline auto make_aead_nonce(
    std::span<const std::uint8_t> iv,
    std::uint64_t sequence) noexcept
    -> std::array<std::uint8_t, 12>;
```

按 TLS 1.3 规范构造 AEAD nonce：将 12 字节 IV 与 64 位序列号按小端序异或，生成每记录唯一的 nonce。

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `iv` | `span<const uint8_t>` | 12 字节初始化向量 |
| `sequence` | `uint64_t` | TLS 记录序列号（从 0 递增） |

#### 返回值

12 字节 nonce 数组。

#### 算法

```
nonce = IV (12 字节)
        XOR
        [0x00, 0x00, 0x00, 0x00] + sequence (小端 8 字节)

示例:
IV       = [01 02 03 04 05 06 07 08 09 0A 0B 0C]
sequence = 0x0000000000000003
padding  = [00 00 00 00 03 00 00 00 00 00 00 00]  (小端)
nonce    = [01 02 03 04 06 06 07 08 09 0A 0B 0C]
```

#### 使用场景

- [[core/stealth/reality/scheme|Reality]] 握手中的 AEAD 加密记录
- [[core/stealth/shadowtls/scheme|ShadowTLS]] 载荷加密
- 任何需要按 TLS 1.3 规范生成 per-record nonce 的伪装方案

### make_record_ad

```cpp
[[nodiscard]] inline auto make_record_ad(
    std::uint16_t encrypted_len) noexcept
    -> std::array<std::uint8_t, 5>;
```

构造 TLS 记录附加数据 (AEAD Additional Authenticated Data)，生成 5 字节 TLS 记录头。

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `encrypted_len` | `uint16_t` | 加密后载荷长度（不含 tag） |

#### 返回值

5 字节附加数据数组，格式如下：

```
┌──────┬──────────────┬──────────────────────┐
│ 0x17 │ 0x03   0x03  │ len_hi    len_lo     │
│ type │ version      │ length               │
└──────┴──────────────┴──────────────────────┘
  1 byte  2 bytes       2 bytes

type    = 0x17 (Application Data)
version = 0x0303 (TLS 1.2 兼容版本号)
length  = encrypted_len (大端序)
```

#### 设计说明

TLS 1.3 使用 TLS 1.2 的版本号 `0x0303` 作为记录层的兼容版本，内容类型固定为 `Application Data (0x17)`。AEAD 加密时，此 5 字节记录头作为 AAD 输入，确保记录头未被篡改。

### xor_with_key

```cpp
inline void xor_with_key(
    std::span<std::byte> data,
    std::span<const std::uint8_t> key) noexcept;
```

用密钥对数据执行循环异或，就地修改 `data`。

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `span<std::byte>` | 待异或的数据（就地修改） |
| `key` | `span<const uint8_t>` | 异或密钥（循环使用） |

#### 算法

```
for i in 0..data.len:
    data[i] ^= key[i % key.len]
```

#### 使用场景

- [[core/stealth/shadowtls/scheme|ShadowTLS]] 载荷混淆：用密码派生的密钥对部分 TLS 握手数据异或
- 任何需要简单载荷混淆的伪装方案

### read_raw_tls_frame

```cpp
auto read_raw_tls_frame(
    net::ip::tcp::socket &sock,
    std::error_code &ec)
    -> net::awaitable<std::optional<memory::vector<std::byte>>>;
```

从 TCP socket 读取一帧完整的原始 TLS 记录。这是**协程函数**，返回 `net::awaitable`。

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `sock` | `tcp::socket &` | TCP socket 引用 |
| `ec` | `error_code &` | 错误码输出参数 |

#### 返回值

- 成功: 包含完整 TLS 帧（记录头 + 载荷）的 `memory::vector<std::byte>`
- 失败: `std::nullopt`（`ec` 被设置为相应错误码）

#### 读取流程

```
TCP Socket
    │
    ▼
1. 读取 5 字节 TLS 记录头
    ┌──────┬──────────────┬──────────────┐
    │ type │ version      │ length       │
    │ 1B   │ 2B           │ 2B           │
    └──────┴──────────────┴──────────────┘
    │
    ▼
2. 解析 length 字段（大端序 uint16_t）
    │
    ▼
3. 读取 length 字节的载荷
    │
    ▼
4. 拼接 返回 [记录头 5B] + [载荷 length B]
    │
    └── 总长度 = 5 + length
```

#### 使用场景

- [[core/stealth/reality/scheme|Reality]] 握手中读取 ServerHello、Certificate 等服务端响应
- [[core/stealth/shadowtls/scheme|ShadowTLS]] 读取真实 TLS 服务器的响应帧
- 任何需要逐帧处理 TLS 数据的伪装方案

## 与伪装方案的集成

```
伪装方案 handshake()
    │
    ├── Reality
    │       ├── read_raw_tls_frame() — 读取远端 TLS 响应
    │       ├── make_aead_nonce() — 构造每记录 nonce
    │       └── make_record_ad() — 构造 AEAD AAD
    │
    ├── ShadowTLS
    │       ├── read_raw_tls_frame() — 读取伪装目标 TLS 响应
    │       ├── xor_with_key() — 载荷混淆
    │       └── make_aead_nonce() — v3 模式加密
    │
    ├── Restls
    │       └── read_raw_tls_frame() — 读取真实 TLS 帧
    │
    └── Native
            └── (不使用 common 工具，直接 Boost.Beast TLS)
```

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[core/memory/container|Memory Container]] | `memory::vector` |
| 依赖 | [[core/protocol/tls/types|TLS Types]] | `hello_features` 等类型 |
| 被依赖 | [[core/stealth/reality/scheme|Reality]] | 握手加密与帧读取 |
| 被依赖 | [[core/stealth/shadowtls/scheme|ShadowTLS]] | 载荷混淆与帧读取 |
| 被依赖 | [[core/stealth/restls/scheme|Restls]] | TLS 帧读取 |

## 性能特性

| 函数 | 分配 | 说明 |
|------|------|------|
| `make_aead_nonce()` | 零分配 | 栈上 `array<uint8_t, 12>` |
| `make_record_ad()` | 零分配 | 栈上 `array<uint8_t, 5>` |
| `xor_with_key()` | 零分配 | 就地修改 |
| `read_raw_tls_frame()` | 一次分配 | `memory::vector<std::byte>` 使用 PMR |

> `make_aead_nonce` 和 `make_record_ad` 是热路径函数，在每个 TLS 记录加密/解密时调用。零分配设计确保不影响吞吐量。

## 相关文档

- [[core/stealth/overview|Stealth 模块总览]] - 伪装方案架构总览
- [[core/stealth/scheme|方案基类]] - stealth_scheme 抽象类
- [[core/stealth/executor|执行器]] - 方案管道执行
- [[core/protocol/tls/types|TLS Types]] - TLS 特征结构定义
- [[core/memory/container|Memory Container]] - PMR 容器定义
- [[core/crypto/aead|AEAD]] - AEAD 加密上下文（使用 common 生成的 nonce/AAD）
