---
layer: core
source: "I:/code/Prism/include/prism/protocol/shadowsocks/framing.hpp"
title: SS2022 协议格式编解码
tags: [protocol, shadowsocks, format, framing, parse, psk, aead]
---

# SS2022 协议格式编解码

> 源码位置: `I:/code/Prism/include/prism/protocol/shadowsocks/framing.hpp`

## 概述

SS2022 协议格式编解码模块，提供地址解析、PSK 解码、加密方法推断等底层解析函数。实现位于 `src/prism/protocol/shadowsocks/framing.cpp`。

## 命名空间

```cpp
namespace psm::protocol::shadowsocks::format
```

## 核心结构

### addr_parse_result

地址解析结果结构，存储从缓冲区解析出的目标地址、端口和字节偏移。

```cpp
struct addr_parse_result
{
    address addr;         // 目标地址
    std::uint16_t port{}; // 目标端口
    std::size_t offset{}; // 地址+端口在缓冲区中占用的总字节数
};
```

> **约束**: `offset` 表示 ATYP+ADDR+PORT 的总字节数，用于在变长头解析中定位后续数据（padding 或 payload）。偏移计算错误会导致后续 AEAD 解密失败。

## 核心函数

### parse_addr_port

从缓冲区解析 SOCKS5 风格地址和端口。内部委托给 `common::framing::parse_ipv4/parse_ipv6/parse_domain` 和 `parse_port`。

```cpp
[[nodiscard]] auto parse_addr_port(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, addr_parse_result>;
```

**解析流程**:
1. 读取 ATYP（1 字节）：0x01=IPv4, 0x03=Domain, 0x04=IPv6
2. 根据 ATYP 调用对应的 `common::framing::parse_*`
3. 解析端口（2 字节大端序）
4. 计算 offset = ATYP(1) + ADDR(var) + PORT(2)

> **约束**: ATYP 不在枚举范围内时返回 `fault::code::unsupported_address`。

### decode_psk

解码 base64 PSK 并验证长度。

```cpp
[[nodiscard]] auto decode_psk(std::string_view base64_psk)
    -> std::pair<fault::code, memory::vector<std::uint8_t>>;
```

**验证规则**:
- 空字符串 → `fault::code::invalid_psk`
- 解码后长度必须为 16 或 32 字节
- 其他长度 → `fault::code::invalid_psk`

### keysalt_len

根据加密方法获取 key/salt 长度。

```cpp
[[nodiscard]] constexpr auto keysalt_len(cipher_method method) noexcept
    -> std::size_t;
```

**映射**: AES-128-GCM → 16, 其他（AES-256/ChaCha20） → 32

### resolve_method

从配置方法字符串解析加密方法。

```cpp
[[nodiscard]] auto resolve_method(std::string_view method_str, std::size_t psk_len) noexcept
    -> cipher_method;
```

**推断逻辑**:
1. 若 `method_str` 非空，精确匹配三个方法名常量
2. 若为空，按 PSK 长度推断：16B → aes_128_gcm, 32B → aes_256_gcm

> **约束**: 32 字节 PSK + 空 method 默认推断为 AES-256-GCM。若实际需要 ChaCha20-Poly1305，必须显式设置 method 字段。

## AEAD 帧结构参考

SS2022 TCP 帧的完整字节布局：

```
请求帧:
┌──────────┬──────────┬─────────────────────────────────────────────────────┐
│ Salt      │ Fixed Header (加密)                                        │
│ (16/32B)  │ type(1) + timestamp(8) + varHeaderLen(2) + AEAD tag(16)    │
├──────────┼──────────┼─────────────────────────────────────────────────────┤
│           │ Variable Header (加密)                                      │
│           │ ATYP(1) + ADDR(var) + PORT(2) + Padding(var) + AEAD tag(16) │
├──────────┼──────────┼─────────────────────────────────────────────────────┤
│           │ 数据流 (AEAD 分帧)                                          │
│           │ [EncLen(2+16)] [EncPayload(N+16)] [EncLen] [EncPayload] ...  │
└──────────┴──────────┴─────────────────────────────────────────────────────┘

响应帧:
┌──────────┬──────────────────────────────────────────────────────────────┐
│ Salt      │ Fixed Header (加密)                                         │
│ (16/32B)  │ type(1) + timestamp(8) + varHeaderLen(2) + AEAD tag(16)    │
├──────────┼──────────────────────────────────────────────────────────────┤
│           │ 数据流 (AEAD 分帧)                                          │
│           │ [EncLen(2+16)] [EncPayload(N+16)] ...                        │
└──────────┴──────────────────────────────────────────────────────────────┘

固定头明文 (11 字节):
┌──────┬───────────┬──────────────┐
│ type │ timestamp │ varHeaderLen │
│ 1B   │ 8B BE     │ 2B BE        │
└──────┴───────────┴──────────────┘

AEAD 长度块 (18 字节):
┌────────────┬──────────┐
│ 长度 (2B)  │ tag (16B)│
└────────────┴──────────┘

数据块:
┌──────────────┬──────────┐
│ payload (NB) │ tag (16B)│
└──────────────┴──────────┘
最大 payload = 0x3FFF (16383 字节)
```

**关键参数**:
- request_type = 0x00, response_type = 0x01
- fixed_header_plain = 11 (type+timestamp+varHeaderLen)
- fixed_header_size = 27 (11 + AEAD tag 16)
- max_chunk_size = 0x3FFF
- len_block_size = 18 (2 + tag 16)
- kdf_context = "shadowsocks 2022 session subkey"

## 使用场景

- **conn::handshake()**: 调用 `parse_addr_port` 解析变长头中的目标地址，调用 `decode_psk` 解码配置中的 PSK
- **conn 构造**: 调用 `resolve_method` 确定加密方法，`keysalt_len` 确定 salt 长度
- **配置加载**: `decode_psk` 在启动时验证 PSK 有效性

## 调用链

- [[core/protocol/shadowsocks/constants|Constants]] - 加密方法枚举定义
- [[core/protocol/common/framing|Framing]] - 底层地址/端口解析
- [[core/protocol/shadowsocks/relay|Relay]] - 中继器使用这些解析函数
- [[core/protocol/shadowsocks/config|Config]] - 配置传递给中继器
- [[core/fault/code|Fault Code]] - 错误码处理
