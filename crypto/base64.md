---
title: "base64 — Base64 编解码"
source: "include/prism/crypto/base64.hpp"
module: "crypto"
type: api
tags: [crypto, base64, 编解码]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[ref/protocol/http-connect|HTTP CONNECT]]"
  - "[[pipeline/protocols/trojan|Trojan]]"
  - "[[stealth/reality/auth|Reality 认证]]"
---

# base64.hpp

> 源码: `include/prism/crypto/base64.hpp`
> 模块: [[crypto|crypto]]

## 概述

Base64 编解码工具。提供轻量级 Base64 编解码函数，实现为 header-only inline 函数，用于 HTTP Basic 认证、SS2022 PSK 解析、Reality 私钥解码等场景。遵循 RFC 4648 标准。

## 命名空间

`psm::crypto`

## 依赖关系

| 依赖 | 用途 | 层级 |
|------|------|------|
| `<string>` | `std::string` 返回值 | 标准库 |
| `<span>` | `std::span` 输入参数 | 标准库 |
| `<string_view>` | `std::string_view` 输入参数 | 标准库 |
| `<array>` | 编解码查找表 | 标准库 |
| `<cctype>` | 空白字符判断 | 标准库 |

## 函数

### base64_encode

功能说明：将原始字节数据编码为标准 Base64 字符串（含 padding）。

签名：

```cpp
[[nodiscard]] inline auto base64_encode(std::span<const std::uint8_t> input) -> std::string;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `input` | `std::span<const std::uint8_t>` | 原始字节数据 |

**返回值**：Base64 编码后的字符串。空输入返回空字符串。

**调用（向下）**：
- `detail::base64_encode_table` — 编码查找表（6 位值到 Base64 字符的编译期映射）
- 纯计算逻辑，无外部库依赖

**被调用（向上）**：
- `protocol::http` — HTTP Basic 认证凭据编码
- `protocol::trojan` — Trojan 协议凭据编码

**涉及的知识域**：
- [[ref/protocol/http-connect|HTTP CONNECT]] — HTTP 代理协议中 Basic 认证的使用场景

---

### base64_decode

功能说明：将 Base64 编码字符串解码为原始数据，支持标准和 URL-safe 变体。

签名：

```cpp
[[nodiscard]] inline auto base64_decode(std::string_view input) -> std::string;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `input` | `std::string_view` | Base64 编码的字符串 |

**返回值**：解码后的原始字符串。以下情况返回空字符串：输入为空、长度非 4 的倍数、padding 超过 2 个、包含非法字符。

**调用（向下）**：
- `detail::base64_decode_table` — 解码查找表（ASCII 字符到 6 位值的编译期映射，无效字符映射为 255）
- `std::isspace()` — 跳过空白字符
- 纯计算逻辑，无外部库依赖

**被调用（向上）**：
- `protocol::http::parser` — 解码 HTTP Basic 认证凭据
- `protocol::shadowsocks::format` — 解码 SS2022 Base64 编码的 PSK
- `stealth::reality::handshake` — 解码 Reality Base64 编码的私钥
- `protocol::vless` — 解码 VLESS 协议中的 Base64 用户 ID

**涉及的知识域**：
- [[ref/protocol/http-connect|HTTP CONNECT]] — HTTP Basic 认证中 Base64 的应用
- [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] — Reality 方案中 Base64 私钥的上下文

---

### detail::make_decode_table

功能说明：在编译期生成 256 字节的 Base64 解码查找表。

签名：

```cpp
constexpr auto make_decode_table() -> std::array<std::uint8_t, 256>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| （无） | — | — |

**返回值**：256 字节查找表，将 ASCII 字符映射到对应的 6 位 Base64 值，无效字符映射为 255。存储于编译期常量 `detail::base64_decode_table`。

**调用（向下）**：
- 纯编译期计算，无运行时依赖

**被调用（向上）**：
- `base64_decode()` — 使用该查找表进行字符到值的映射

**涉及的知识域**：
- [[ref/programming/constexpr|constexpr]] — 编译期计算与常量表达式

---

### detail::base64_encode_table

功能说明：编译期常量编码查找表，将 6 位值映射到 Base64 字符。

签名：

```cpp
constexpr char base64_encode_table[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
```

| 参数 | 类型 | 说明 |
|------|------|------|
| （无） | — | — |

**返回值**：64 字符的编码查找表常量。

**调用（向下）**：
- 纯数据常量，无运行时依赖

**被调用（向上）**：
- `base64_encode()` — 使用该查找表进行值到字符的映射

**涉及的知识域**：
- [[ref/programming/constexpr|constexpr]] — 编译期计算与常量表达式
