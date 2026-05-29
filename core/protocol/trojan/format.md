---
layer: core
source: "include/prism/protocol/trojan/framing.hpp"
title: Trojan 协议格式编解码
tags: [protocol, trojan, format, framing, credential, udp, parse]
---

# Trojan 协议格式编解码

> 源码位置: `include/prism/protocol/trojan/framing.hpp`

## 概述

Trojan 协议格式编解码模块，提供凭据解码、CRLF 验证、命令/地址类型解析和 UDP 帧编解码。实现位于 `src/prism/protocol/trojan/framing.cpp`。地址/端口解析委托给 `common::framing`。

## 命名空间

```cpp
namespace psm::protocol::trojan::format
```

## 核心结构

### header_parse

```cpp
struct header_parse
{
    command cmd;       // 命令类型
    address_type atyp; // 地址类型
};
```

### udp_routed

```cpp
struct udp_routed
{
    address destination_address;
    std::uint16_t destination_port;
};
```

### udp_parse_result

```cpp
struct udp_parse_result
{
    address destination_address;
    std::uint16_t destination_port{};
    std::size_t payload_offset{};
    std::size_t payload_size{};
};
```

## Trojan 请求字节布局

```
Trojan TCP 请求:
┌────────────────────┬──────┬──────┬──────┬──────────────────┬──────────┐
│ CREDENTIAL (56B)   │ CRLF │ CMD  │ ATYP │ ADDR + PORT      │ Payload  │
│ SHA224(password)   │ \r\n │ 1B   │ 1B   │ 变长             │ 变长     │
│ hex 明文            │      │      │      │                  │          │
└────────────────────┴──────┴──────┴──────┴──────────────────┴──────────┘

CREDENTIAL 生成算法:
  SHA224(password) → 28 字节二进制 → 转 56 字节 hex 字符串
  示例: echo -n "password" | openssl dgst -sha224

CMD 值: 0x01=CONNECT, 0x03=UDP_ASSOCIATE, 0x7F=MUX
ATYP 值: 0x01=IPv4, 0x03=Domain, 0x04=IPv6
```

```
Trojan UDP 帧:
┌──────┬──────────────────┬──────────┬──────┬──────────┐
│ ATYP │ ADDR + PORT      │ Length   │ CRLF │ Payload  │
│ 1B   │ 变长             │ 2B BE    │ \r\n │ N 字节   │
└──────┴──────────────────┴──────────┴──────┴──────────┘
最小长度: ATYP(1) + IPv4(4) + PORT(2) + Length(2) + CRLF(2) = 11 字节
```

## 核心函数

### parse_credential

```cpp
auto parse_credential(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, std::array<char, 56>>;
```

验证 56 字节凭据是否为合法 hex 字符串（0-9, a-f, A-F）。返回 `fault::code::protocol_error` 若包含非法字符。

> **约束**: 密码以 SHA224 hex 明文在 TLS 内层传输。TLS 配置错误（如降级到明文）时密码直接暴露。

### parse_crlf

```cpp
auto parse_crlf(std::span<const std::uint8_t> buffer) -> fault::code;
```

验证 `\r\n` 分隔符。

### parse_cmd_atyp

```cpp
auto parse_cmd_atyp(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, header_parse>;
```

解析命令和地址类型（各 1 字节）。

### UDP 编解码

```cpp
// 构建: ATYP + ADDR + PORT + Length(2B BE) + CRLF + Payload
auto build_udp_pkt(const udp_routed &frame, std::span<const std::byte> payload,
                   memory::vector<std::byte> &out) -> fault::code;

// 解析: 反向提取地址、端口和 payload
auto parse_udp_pkt(std::span<const std::byte> buffer)
    -> std::pair<fault::code, udp_parse_result>;
```

**与 VLESS UDP 的区别**: Trojan UDP 帧包含 Length+CRLF 字段，VLESS 无。

## 实现边界

- **56 字节 SHA224 明文传输**: TLS 配置错误（如降级）时密码暴露
- **缓冲区精确边界**: 域地址长度 255 时 `required_total = 320`，恰好等于固定缓冲区大小，零余量
- **日志不区分错误类型**: `credential verification failed` 不区分"格式错误"还是"密码错误"

排障建议: `echo -n "password" | openssl dgst -sha224` 验证客户端密码配置

详见 [[dev/debugging/deep-dive/protocol-boundaries|代理协议实现边界与认证深层分析]]

## 调用链

- [[core/protocol/trojan/constants|Constants]] - 命令和地址类型枚举定义
- [[core/protocol/common/framing|Framing]] - 共享地址/端口解析
- [[core/protocol/trojan/relay|Relay]] - 中继器使用这些解析函数
- [[core/fault/code|Fault Code]] - 错误码处理
