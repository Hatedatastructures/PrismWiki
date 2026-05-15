---
title: "wire — SOCKS5 协议线级解析"
source: "include/prism/protocol/socks5/wire.hpp"
module: "protocol"
type: api
tags: [protocol, socks5, wire, 解析, 编解码, RFC1928, 零拷贝]
related:
  - "[[protocol/socks5/constants|constants]]"
  - "[[protocol/socks5/stream|stream]]"
  - "[[protocol/socks5/message|message]]"
  - "[[protocol/common/address|address]]"
created: 2026-05-15
updated: 2026-05-15
---

# wire.hpp

> 源码: `include/prism/protocol/socks5/wire.hpp`
> 模块: [[protocol|protocol]] > socks5

## 概述

SOCKS5 协议线级解析 (RFC 1928)。提供 SOCKS5 协议报文的底层解析函数，包括头部、IPv4、IPv6、域名、端口和 UDP 数据报的编解码。所有函数设计为零拷贝友好，直接操作字节缓冲区。函数返回值使用 `fault::code` 错误码系统。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[protocol/socks5/constants|constants]] | 命令字、地址类型枚举 |
| 依赖 | [[protocol/socks5/message|message]] | 地址类型定义 |
| 依赖 | [[memory|memory]] | `memory::vector` |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[protocol/socks5/stream|stream]] | SOCKS5 中继器调用解析函数 |

## 命名空间

`psm::protocol::socks5::wire`

---

## 结构体: header_parse

SOCKS5 请求头部解析结果。存储解析后的 4 字节头部字段。

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | `std::uint8_t` | 协议版本，SOCKS5 固定为 `0x05` |
| `cmd` | `command` | 命令类型（connect/bind/udp_associate） |
| `rsv` | `std::uint8_t` | 保留字段，必须为 `0x00` |
| `atyp` | `address_type` | 地址类型（ipv4/domain/ipv6） |

---

## 结构体: udp_header

SOCKS5 UDP 数据报头部，遵循 RFC 1928 Section 7。RSV 必须为 `0x0000`，FRAG=0 表示独立数据报。本项目仅支持 FRAG=0。

| 字段 | 类型 | 说明 |
|------|------|------|
| `destination_address` | `address` | 目标地址（variant 类型） |
| `destination_port` | `std::uint16_t` | 目标端口（主机字节序） |
| `frag` | `std::uint8_t` | 分片序号，0 表示独立数据报 |

## 结构体: udp_header_parse

| 字段 | 类型 | 说明 |
|------|------|------|
| `header` | `udp_header` | UDP 报头信息 |
| `header_size` | `std::size_t` | 头部字节数，用于定位 DATA 起始位置 |

---

## 结构体: password_auth_request

RFC 1929 用户名/密码认证请求。

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | `std::uint8_t` | 子协商版本，固定 `0x01` |
| `username` | `std::string_view` | 用户名（1-255 字节） |
| `password` | `std::string_view` | 密码（1-255 字节） |

---

## 函数: parse_header()

### 功能说明

解析 4 字节 SOCKS5 请求头部。验证版本是否为 `0x05`。

### 签名

```cpp
inline auto parse_header(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, header_parse>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 输入字节数组 |

### 返回值

`std::pair<fault::code, header_parse>` — 错误码和解析后的头部。

### 调用（向下）

无（纯数据解析）

### 被调用（向上）

- [[protocol/socks5/stream|stream]] `read_request_header()` 调用

### 知识域

RFC 1928 SOCKS5 协议格式

---

## 函数: parse_ipv4()

### 功能说明

从缓冲区读取 4 字节 IPv4 地址，网络字节序存储。

### 签名

```cpp
inline auto parse_ipv4(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv4_address>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 至少 4 字节 |

### 返回值

`std::pair<fault::code, ipv4_address>` — 错误码和 IPv4 地址。

### 调用（向下）

`std::memcpy` 拷贝 4 字节

### 被调用（向上）

- [[protocol/socks5/stream|stream]] `read_address<4>()`
- `decode_udp_header()` UDP 报头 IPv4 解析

### 知识域

IPv4 地址格式

---

## 函数: parse_ipv6()

### 功能说明

从缓冲区读取 16 字节 IPv6 地址。

### 签名

```cpp
inline auto parse_ipv6(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv6_address>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 至少 16 字节 |

### 返回值

`std::pair<fault::code, ipv6_address>` — 错误码和 IPv6 地址。

### 调用（向下）

`std::memcpy` 拷贝 16 字节

### 被调用（向上）

- [[protocol/socks5/stream|stream]] `read_address<16>()`

### 知识域

IPv6 地址格式

---

## 函数: parse_domain()

### 功能说明

解析 SOCKS5 域名格式：LEN(1) + DOMAIN(n)。最大 255 字节。

### 签名

```cpp
inline auto parse_domain(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, domain_address>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 域名数据 |

### 返回值

`std::pair<fault::code, domain_address>` — 错误码和域名地址。

### 调用（向下）

`std::memcpy` 拷贝域名内容

### 被调用（向上）

- [[protocol/socks5/stream|stream]] `read_domain_address()`

### 知识域

SOCKS5 域名编码

---

## 函数: decode_port()

### 功能说明

读取 2 字节大端序端口，转换为主机字节序。

### 签名

```cpp
inline auto decode_port(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, uint16_t>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 至少 2 字节 |

### 返回值

`std::pair<fault::code, uint16_t>` — 错误码和端口值。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/socks5/stream|stream]] 地址读取函数

### 知识域

大端序编码

---

## 函数: encode_udp_header()

### 功能说明

将 UDP 报头编码为 SOCKS5 格式（不含 DATA）。预分配最多 262 字节。

### 签名

```cpp
inline auto encode_udp_header(const udp_header &header, memory::vector<std::uint8_t> &out)
    -> fault::code;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `header` | `const udp_header &` | UDP 报头 |
| `out` | `memory::vector<std::uint8_t> &` | 输出缓冲区 |

### 返回值

`fault::code` — 编码结果。

### 调用（向下）

`memory::vector::push_back()` / `insert()`

### 被调用（向上）

- `encode_udp_datagram()` 完整数据报编码
- [[protocol/socks5/stream|stream]] 构建 UDP 响应

### 知识域

RFC 1928 Section 7 UDP 格式

---

## 函数: decode_udp_header()

### 功能说明

解析 SOCKS5 UDP 报头。验证 RSV=0x0000、FRAG=0。支持 IPv4/IPv6/域名。

### 签名

```cpp
inline auto decode_udp_header(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, udp_header_parse>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | UDP 数据报 |

### 返回值

`std::pair<fault::code, udp_header_parse>` — 错误码和解析结果。

### 调用（向下）

`parse_ipv4()` / `parse_ipv6()` / `parse_domain()` / `decode_port()`

### 被调用（向上）

- [[protocol/socks5/stream|stream]] `relay_single_datagram()` 解析入站数据报

### 知识域

SOCKS5 UDP 数据报格式

---

## 函数: encode_udp_datagram()

### 功能说明

编码完整 SOCKS5 UDP 数据报（报头 + 用户数据）。

### 签名

```cpp
inline auto encode_udp_datagram(const udp_header &header, std::span<const std::uint8_t> data, memory::vector<std::uint8_t> &out)
    -> fault::code;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `header` | `const udp_header &` | UDP 报头 |
| `data` | `std::span<const std::uint8_t>` | 用户数据 |
| `out` | `memory::vector<std::uint8_t> &` | 输出缓冲区 |

### 返回值

`fault::code` — 编码结果。

### 调用（向下）

`encode_udp_header()`

### 被调用（向上）

- [[protocol/socks5/stream|stream]] `relay_single_datagram()`

### 知识域

SOCKS5 UDP 数据报格式

---

## 函数: parse_password_auth()

### 功能说明

解析 RFC 1929 认证请求：VER(1) + ULEN(1) + UNAME(n) + PLEN(1) + PASSWD(n)。验证版本为 0x01。

### 签名

```cpp
[[nodiscard]] inline auto parse_password_auth(const std::span<const std::uint8_t> data)
    -> std::pair<fault::code, password_auth_request>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `std::span<const std::uint8_t>` | 完整认证请求数据 |

### 返回值

`std::pair<fault::code, password_auth_request>` — 错误码和认证请求。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/socks5/stream|stream]] `perform_password_auth()`

### 知识域

RFC 1929 认证

---

## 函数: build_password_auth_response()

### 功能说明

构建 RFC 1929 认证响应：VER(0x01) + STATUS(0x00=成功/0x01=失败)。

### 签名

```cpp
[[nodiscard]] inline auto build_password_auth_response(const bool success)
    -> std::array<std::uint8_t, 2>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `success` | `const bool` | 认证是否成功 |

### 返回值

`std::array<std::uint8_t, 2>` — 2 字节响应。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/socks5/stream|stream]] `perform_password_auth()`

### 知识域

RFC 1929 认证响应格式

## 相关页面

- [[protocol/socks5/constants|constants]] — 协议常量
- [[protocol/socks5/stream|stream]] — SOCKS5 中继器
- [[protocol/socks5/message|message]] — 消息结构
- [[protocol/common/address|address]] — 共享地址类型
