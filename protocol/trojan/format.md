---
title: "format — Trojan 协议格式编解码"
source: "include/prism/protocol/trojan/format.hpp"
module: "protocol"
type: api
tags: [protocol, trojan, format, 编解码, UDP, 凭据]
related:
  - "[[protocol/trojan/constants|constants]]"
  - "[[protocol/trojan/relay|relay]]"
  - "[[protocol/trojan/message|message]]"
  - "[[protocol/common/address|address]]"
created: 2026-05-15
updated: 2026-05-15
---

# format.hpp

> 源码: `include/prism/protocol/trojan/format.hpp`
> 实现: `src/prism/protocol/trojan/format.cpp`
> 模块: [[protocol|protocol]] > trojan

## 概述

Trojan 协议格式编解码。提供凭据解码、CRLF 验证、命令解析、地址解析等底层函数声明。函数实现位于 `format.cpp` 中。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[protocol/trojan/constants|constants]] | 命令字、地址类型枚举 |
| 依赖 | [[protocol/trojan/message|message]] | 地址类型定义 |
| 依赖 | [[memory|memory]] | `memory::vector` |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[protocol/trojan/relay|relay]] | Trojan 中继器调用解析函数 |

## 命名空间

`psm::protocol::trojan::format`

---

## 结构体: header_parse

协议头部解析结果，存储从协议头部解析出的命令和地址类型。

| 字段 | 类型 | 说明 |
|------|------|------|
| `cmd` | `command` | 命令类型（connect/udp_associate/mux） |
| `atyp` | `address_type` | 地址类型（ipv4/domain/ipv6） |

---

## 结构体: udp_frame

Trojan UDP 帧信息，描述一个 Trojan UDP 数据包的目标地址和端口。

| 字段 | 类型 | 说明 |
|------|------|------|
| `destination_address` | `address` | 目标地址（variant 类型） |
| `destination_port` | `std::uint16_t` | 目标端口 |

---

## 结构体: udp_parse_result

Trojan UDP 数据包解析结果，包含解析出的目标地址、端口以及 payload 偏移和大小。

| 字段 | 类型 | 说明 |
|------|------|------|
| `destination_address` | `address` | 目标地址 |
| `destination_port` | `std::uint16_t` | 目标端口 |
| `payload_offset` | `std::size_t` | payload 在缓冲区中的偏移 |
| `payload_size` | `std::size_t` | payload 大小 |

---

## 函数: parse_credential()

### 功能说明

解析用户凭据。从缓冲区读取 56 字节的 SHA224 哈希值。

### 签名

```cpp
auto parse_credential(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, std::array<char, 56>>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 包含凭据的缓冲区，至少 56 字节 |

### 返回值

`std::pair<fault::code, std::array<char, 56>>` — 错误码和凭据数组（56 字节 SHA224 哈希）。

### 调用（向下）

无（纯数据解析）

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `handshake()` 读取凭据

### 知识域

SHA224 哈希、Trojan 认证

---

## 函数: parse_crlf()

### 功能说明

验证 CRLF 分隔符。Trojan 协议头部中凭据和命令之间使用 `\r\n` 分隔。

### 签名

```cpp
auto parse_crlf(const std::span<const std::uint8_t> buffer) -> fault::code;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 包含 CRLF 的缓冲区，至少 2 字节 |

### 返回值

`fault::code` — 验证结果错误码。`success` 表示有效 CRLF。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `handshake()` 验证分隔符

### 知识域

Trojan 协议格式

---

## 函数: parse_cmd_atyp()

### 功能说明

解析命令和地址类型。从 2 字节缓冲区中读取命令字和地址类型。

### 签名

```cpp
auto parse_cmd_atyp(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, header_parse>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 至少 2 字节（CMD + ATYP） |

### 返回值

`std::pair<fault::code, header_parse>` — 错误码和解析结果。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `handshake()` 解析命令

### 知识域

Trojan 协议头部格式

---

## 函数: parse_ipv4()

### 功能说明

解析 IPv4 地址，从缓冲区读取 4 字节。

### 签名

```cpp
auto parse_ipv4(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv4_address>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 至少 4 字节 |

### 返回值

`std::pair<fault::code, ipv4_address>` — 错误码和 IPv4 地址。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `handshake()` 解析地址

### 知识域

IPv4 地址格式

---

## 函数: parse_ipv6()

### 功能说明

解析 IPv6 地址，从缓冲区读取 16 字节。

### 签名

```cpp
auto parse_ipv6(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv6_address>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 至少 16 字节 |

### 返回值

`std::pair<fault::code, ipv6_address>` — 错误码和 IPv6 地址。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `handshake()` 解析地址

### 知识域

IPv6 地址格式

---

## 函数: parse_domain()

### 功能说明

解析域名地址。格式为长度字节加域名内容。

### 签名

```cpp
auto parse_domain(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, domain_address>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 域名数据 |

### 返回值

`std::pair<fault::code, domain_address>` — 错误码和域名地址。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `handshake()` 解析地址

### 知识域

域名编码格式

---

## 函数: parse_port()

### 功能说明

解析端口号，从缓冲区读取 2 字节大端序整数。

### 签名

```cpp
auto parse_port(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, uint16_t>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 至少 2 字节 |

### 返回值

`std::pair<fault::code, uint16_t>` — 错误码和端口号。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `handshake()` 解析端口

### 知识域

大端序编码

---

## 函数: build_udp_packet()

### 功能说明

构建 Trojan UDP 数据包（mihomo 兼容格式）。

### 签名

```cpp
auto build_udp_packet(const udp_frame &frame, std::span<const std::byte> payload,
                      memory::vector<std::byte> &out) -> fault::code;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `frame` | `const udp_frame &` | UDP 帧信息 |
| `payload` | `std::span<const std::byte>` | 用户数据 |
| `out` | `memory::vector<std::byte> &` | 输出缓冲区 |

### 返回值

`fault::code` — 编码结果。

### 调用（向下）

- `memory::vector::insert()` 写入数据

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `udp_frame_loop()` 构建响应

### 知识域

Trojan UDP 帧格式

---

## 函数: parse_udp_packet()

### 功能说明

解析 Trojan UDP 数据包（mihomo 兼容格式）。提取目标地址、端口和 payload 偏移。

### 签名

```cpp
auto parse_udp_packet(std::span<const std::byte> buffer)
    -> std::pair<fault::code, udp_parse_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | UDP 数据包缓冲区 |

### 返回值

`std::pair<fault::code, udp_parse_result>` — 错误码和解析结果。

### 调用（向下）

- 内部地址解析

### 被调用（向上）

- [[protocol/trojan/relay|relay]] `udp_frame_loop()` 解析入站数据

### 知识域

Trojan UDP 帧格式

## 相关页面

- [[protocol/trojan/constants|constants]] — 协议常量
- [[protocol/trojan/relay|relay]] — 协议中继器
- [[protocol/trojan/message|message]] — 消息结构
