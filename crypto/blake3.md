---
title: "blake3 — BLAKE3 密钥派生"
source: "include/prism/crypto/blake3.hpp"
module: "crypto"
type: api
tags: [crypto, blake3, 密钥派生, 哈希, ss2022, sip022]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[ref/crypto/blake3|BLAKE3 算法]]"
  - "[[pipeline/protocols/shadowsocks|Shadowsocks 协议]]"
  - "[[crypto/hkdf|HKDF 密钥派生]]"
---

# blake3.hpp

> 源码: `include/prism/crypto/blake3.hpp`
> 实现: `src/prism/crypto/blake3.cpp`
> 模块: [[crypto|crypto]]

## 概述

BLAKE3 密钥派生工具。提供 BLAKE3 `derive_key` 功能，用于 SS2022 (SIP022) 会话密钥派生。包装 BLAKE3 C API，提供类型安全的 C++ 接口。函数命名为 `derive_key`（非 `blake3_derive_key`）以避免与 C API 冲突。

BLAKE3 的 derive_key 模式比 HKDF 更快（BLAKE3 本身就比 SHA-256 快很多），且不需要"提取"和"扩展"两步。

## 命名空间

`psm::crypto`

## 依赖关系

| 依赖 | 用途 | 层级 |
|------|------|------|
| `<blake3.h>` | BLAKE3 C API | 外部 (FetchContent) |
| `<span>` | `std::span` 输入参数 | 标准库 |
| `<string_view>` | `std::string_view` 上下文 | 标准库 |
| `<vector>` | `std::vector` 返回值 | 标准库 |

## 函数

### derive_key() — 写入外部缓冲区

使用 BLAKE3 的 derive_key 模式，从上下文字符串和密钥材料派生指定长度的密钥，结果写入调用方提供的缓冲区。

```cpp
auto derive_key(std::string_view context,
                std::span<const std::uint8_t> material,
                std::size_t out_len,
                std::span<std::uint8_t> out)
    -> void;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `context` | `std::string_view` | 上下文字符串，用于域分离（如 `"shadowsocks 2022 session subkey"`） |
| `material` | `std::span<const std::uint8_t>` | 输入密钥材料（如主密钥与 salt 拼接） |
| `out_len` | `std::size_t` | 输出密钥长度（字节数，BLAKE3 支持任意长度） |
| `out` | `std::span<std::uint8_t>` | 输出缓冲区，必须至少 `out_len` 字节 |

- **返回值**: 无（`void`）
- **调用（向下）**: `blake3_hasher_init_derive_key_raw()` (BLAKE3 C API) -> `blake3_hasher_update()` -> `blake3_hasher_finalize()`
- **被调用（向上）**: `derive_key(vector 版本)` 内部转发调用; `shadowsocks::relay` 派生会话子密钥; `shadowsocks::tracker` 派生 header 加密密钥; `CryptoBench` 基准测试
- **涉及的知识域**: [[ref/crypto/blake3|BLAKE3 算法]], [[ref/crypto/aes-gcm|AES-GCM]], [[ref/anti-censorship/replay-attack|重放攻击防护]]

---

### derive_key() — 返回 vector 版本

使用 BLAKE3 的 derive_key 模式派生密钥，自动分配输出缓冲区并返回结果 vector。

```cpp
[[nodiscard]] auto derive_key(std::string_view context,
                              std::span<const std::uint8_t> material,
                              std::size_t out_len)
    -> std::vector<std::uint8_t>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `context` | `std::string_view` | 上下文字符串，用于域分离 |
| `material` | `std::span<const std::uint8_t>` | 输入密钥材料 |
| `out_len` | `std::size_t` | 输出密钥长度（字节数） |

- **返回值**: `std::vector<std::uint8_t>` — 包含派生密钥的字节向量
- **调用（向下）**: `derive_key()`（写入外部缓冲区版本）
- **被调用（向上）**: `shadowsocks::relay` 派生会话子密钥; `shadowsocks::tracker` 派生 header 加密密钥; `CryptoBench` 基准测试; `Blake3` 单元测试
- **涉及的知识域**: [[ref/crypto/blake3|BLAKE3 算法]], [[ref/crypto/hkdf|HKDF 对比]], [[pipeline/protocols/shadowsocks|Shadowsocks 协议]]

---

## 调用链

```
向上调用者:
  shadowsocks::relay    → derive_key()  派生 SS2022 会话子密钥 (加密 payload)
  shadowsocks::tracker  → derive_key()  派生 SS2022 header 加密密钥
  CryptoBench           → derive_key()  基准测试 BLAKE3 性能
  Blake3 测试           → derive_key()  验证确定性、域隔离、可变长度

向下依赖:
  derive_key(→void)     → blake3_hasher_init_derive_key_raw()  初始化 derive_key 模式
                        → blake3_hasher_update()               输入密钥材料
                        → blake3_hasher_finalize()             输出任意长度密钥

  derive_key(→vector)   → derive_key(→void)  转发调用
```

## 知识域

- [[ref/crypto/blake3|BLAKE3 算法]] — BLAKE3 哈希函数原理与安全性
- [[ref/crypto/hkdf|HKDF 密钥派生]] — 替代的密钥派生方案（提取 + 扩展两步）
- [[pipeline/protocols/shadowsocks|Shadowsocks 协议]] — SS2022 中 BLAKE3 的实际应用场景
- [[crypto/aead|AEAD 加密]] — 派生密钥的下游消费者
