---
layer: core
source: "I:/code/Prism/include/prism/protocol/shadowsocks/format.hpp"
title: SS2022 协议格式编解码
---

# SS2022 协议格式编解码

> 源码位置: `I:/code/Prism/include/prism/protocol/shadowsocks/format.hpp`

## 概述

SS2022 协议格式编解码模块，提供地址解析、PSK 解码等底层解析函数声明。

## 命名空间

```cpp
namespace psm::protocol::shadowsocks::format
```

## 核心结构

### address_parse_result

地址解析结果结构，存储从缓冲区解析出的目标地址、端口和字节偏移。

```cpp
struct address_parse_result
{
    address addr;         // 目标地址
    std::uint16_t port{}; // 目标端口
    std::size_t offset{}; // 地址+端口在缓冲区中占用的总字节数
};
```

## 核心函数

### parse_address_port

从缓冲区解析 SOCKS5 风格地址和端口。

```cpp
[[nodiscard]] auto parse_address_port(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, address_parse_result>;
```

**参数**:
- `buffer`: 包含 ATYP+ADDR+PORT 的缓冲区

**返回**: 错误码和解析结果

### decode_psk

解码 base64 PSK 并验证长度。

```cpp
[[nodiscard]] auto decode_psk(std::string_view base64_psk)
    -> std::pair<fault::code, std::vector<std::uint8_t>>;
```

**参数**:
- `base64_psk`: Base64 编码的 PSK 字符串

**返回**: 错误码和原始 PSK 字节（16 或 32 字节）

### key_salt_length

根据加密方法获取 key/salt 长度。

```cpp
[[nodiscard]] constexpr auto key_salt_length(cipher_method method) noexcept
    -> std::size_t
{
    return method == cipher_method::aes_128_gcm ? 16 : 32;
}
```

**参数**:
- `method`: 加密方法枚举

**返回**: 密钥或 salt 的字节长度（AES-128 为 16，其余为 32）

### resolve_cipher_method

从配置方法字符串解析加密方法。

```cpp
[[nodiscard]] auto resolve_cipher_method(std::string_view method_str, std::size_t psk_len) noexcept
    -> cipher_method;
```

**参数**:
- `method_str`: 方法名字符串（可为空，自动推断）
- `psk_len`: PSK 字节长度（用于自动推断）

**返回**: 加密方法枚举

## 调用链

- [[core/protocol/shadowsocks/constants|Constants]] - 加密方法枚举定义
- [[core/protocol/shadowsocks/relay|Relay]] - 中继器使用这些解析函数
- [[core/protocol/shadowsocks/config|Config]] - 配置传递给中继器
- [[core/fault/code|Fault Code]] - 错误码处理