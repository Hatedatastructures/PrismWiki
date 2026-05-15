---
title: "format — SS2022 协议格式编解码"
source: "include/prism/protocol/shadowsocks/format.hpp"
module: "protocol"
type: api
tags: [protocol, shadowsocks, format, 编解码, 地址解析, PSK, cipher]
related:
  - "[[protocol/shadowsocks/constants|constants]]"
  - "[[protocol/shadowsocks/relay|relay]]"
  - "[[protocol/shadowsocks/message|message]]"
  - "[[protocol/common/address|address]]"
created: 2026-05-15
updated: 2026-05-15
---

# format.hpp

> 源码: `include/prism/protocol/shadowsocks/format.hpp`
> 实现: `src/prism/protocol/shadowsocks/format.cpp`
> 模块: [[protocol|protocol]] > shadowsocks

## 概述

SS2022 协议格式编解码。提供地址解析、PSK 解码、加密方法推断等底层解析函数声明。函数实现位于 `format.cpp` 中。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[protocol/shadowsocks/constants|constants]] | 加密方法枚举、地址类型 |
| 依赖 | [[protocol/shadowsocks/message|message]] | 地址类型定义 |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[protocol/shadowsocks/relay|relay]] | SS2022 中继器调用解析函数 |

## 命名空间

`psm::protocol::shadowsocks::format`

---

## 结构体: address_parse_result

地址解析结果，存储从缓冲区解析出的目标地址、端口和字节偏移。

| 字段 | 类型 | 说明 |
|------|------|------|
| `addr` | `address` | 目标地址（variant 类型） |
| `port` | `std::uint16_t` | 目标端口 |
| `offset` | `std::size_t` | 地址+端口在缓冲区中占用的总字节数 |

---

## 函数: parse_address_port()

### 功能说明

从缓冲区解析 SOCKS5 风格地址和端口。地址格式为 ATYP(1) + ADDR(变长) + PORT(2)。支持 IPv4(0x01)、域名(0x03)、IPv6(0x04) 三种地址类型。

### 签名

```cpp
[[nodiscard]] auto parse_address_port(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, address_parse_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 包含 ATYP+ADDR+PORT 的缓冲区 |

### 返回值

`std::pair<fault::code, address_parse_result>` — 错误码和解析结果。

### 调用（向下）

- 内部地址解析逻辑

### 被调用（向上）

- [[protocol/shadowsocks/relay|relay]] `read_variable_header()` 解析目标地址

### 知识域

SOCKS5 地址格式、SS2022 协议

---

## 函数: decode_psk()

### 功能说明

解码 Base64 编码的 PSK 并验证长度。解码后必须为 16 字节（AES-128-GCM）或 32 字节（AES-256-GCM/ChaCha20-Poly1305）。

### 签名

```cpp
[[nodiscard]] auto decode_psk(std::string_view base64_psk)
    -> std::pair<fault::code, std::vector<std::uint8_t>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `base64_psk` | `std::string_view` | Base64 编码的 PSK 字符串 |

### 返回值

`std::pair<fault::code, std::vector<std::uint8_t>>` — 错误码和原始 PSK 字节。

### 调用（向下）

- Base64 解码

### 被调用（向上）

- [[protocol/shadowsocks/relay|relay]] 构造时解码 PSK

### 知识域

Base64 编码、密钥管理

---

## 函数: key_salt_length()

### 功能说明

根据加密方法获取 key/salt 长度。AES-128-GCM 返回 16，其余返回 32。

### 签名

```cpp
[[nodiscard]] constexpr auto key_salt_length(cipher_method method) noexcept -> std::size_t;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `method` | `cipher_method` | 加密方法枚举 |

### 返回值

`std::size_t` — 密钥或 salt 的字节长度。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/shadowsocks/relay|relay]] 确定 salt 长度

### 知识域

AEAD 加密参数

---

## 函数: resolve_cipher_method()

### 功能说明

从配置方法字符串解析加密方法。支持显式指定方法名或根据 PSK 长度自动推断：16 字节 -> AES-128-GCM，32 字节 -> AES-256-GCM。显式设置时支持 `"2022-blake3-chacha20-poly1305"`。

### 签名

```cpp
[[nodiscard]] auto resolve_cipher_method(std::string_view method_str, std::size_t psk_len) noexcept
    -> cipher_method;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `method_str` | `std::string_view` | 方法名字符串（可为空，自动推断） |
| `psk_len` | `std::size_t` | PSK 字节长度（用于自动推断） |

### 返回值

`cipher_method` — 加密方法枚举。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/shadowsocks/relay|relay]] 构造时确定加密方法

### 知识域

SS2022 加密方法选择

## 相关页面

- [[protocol/shadowsocks/constants|constants]] — 协议常量
- [[protocol/shadowsocks/relay|relay]] — 协议中继器
- [[protocol/shadowsocks/message|message]] — 消息结构
