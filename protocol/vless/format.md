---
title: "format — VLESS 协议格式编解码"
source: "include/prism/protocol/vless/format.hpp"
module: "protocol"
type: api
tags: [protocol, vless, format, 编解码, UDP, 响应]
related:
  - "[[protocol/vless/constants|constants]]"
  - "[[protocol/vless/relay|relay]]"
  - "[[protocol/vless/message|message]]"
  - "[[protocol/common/address|address]]"
created: 2026-05-15
updated: 2026-05-15
---

# format.hpp

> 源码: `include/prism/protocol/vless/format.hpp`
> 实现: `src/prism/protocol/vless/format.cpp`
> 模块: [[protocol|protocol]] > vless

## 概述

VLESS 协议格式编解码。提供请求解析和响应序列化函数声明。函数实现位于 `format.cpp` 中。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[protocol/vless/constants|constants]] | 版本号、命令字、地址类型 |
| 依赖 | [[protocol/vless/message|message]] | 地址类型定义 |
| 依赖 | [[memory|memory]] | `memory::vector` |
| 依赖 | [[fault|fault]] | 错误码 |
| 被依赖 | [[protocol/vless/relay|relay]] | VLESS 中继器调用解析函数 |

## 命名空间

`psm::protocol::vless::format`

---

## 结构体: udp_frame

VLESS UDP 帧信息，描述一个 VLESS UDP 数据包的目标地址和端口。

| 字段 | 类型 | 说明 |
|------|------|------|
| `destination_address` | `address` | 目标地址（variant 类型） |
| `destination_port` | `std::uint16_t` | 目标端口 |

---

## 结构体: udp_parse_result

VLESS UDP 数据包解析结果，包含从 TLS 流中读取的 UDP 数据包解析出的目标地址、端口以及 payload 偏移和大小。

| 字段 | 类型 | 说明 |
|------|------|------|
| `destination_address` | `address` | 目标地址 |
| `destination_port` | `std::uint16_t` | 目标端口 |
| `payload_offset` | `std::size_t` | payload 在缓冲区中的偏移 |
| `payload_size` | `std::size_t` | payload 大小 |

---

## 函数: parse_request()

### 功能说明

从 wire buffer 解析 VLESS 请求头。格式为 `[Version 1B][UUID 16B][AddnlInfoLen 1B][AddnlInfo var][Cmd 1B][Port 2B BE][Atyp 1B][Addr var]`。仅支持 plain VLESS（AddnlInfoLen = 0）。

### 签名

```cpp
auto parse_request(std::span<const std::uint8_t> buffer) -> std::optional<request>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::uint8_t>` | 包含完整 VLESS 请求头的缓冲区 |

### 返回值

`std::optional<request>` — 解析成功返回请求结构，失败返回 `std::nullopt`。

### 调用（向下）

- [[protocol/vless/message|message]] `request` 结构构造

### 被调用（向上）

- [[protocol/vless/relay|relay]] `handshake()` 解析请求

### 知识域

VLESS 协议格式

---

## 函数: make_response()

### 功能说明

返回 `[Version 0x00][Addons Length 0x00]` 的 2 字节响应数组。客户端（mihomo/Xray/sing-box）期望读取 2 字节响应，仅发送 1 字节会导致客户端将后续 smux ACK 数据误读为 Addons Length，造成流偏移。

### 签名

```cpp
[[nodiscard]] constexpr auto make_response() -> std::array<std::byte, 2>;
```

### 参数

无

### 返回值

`std::array<std::byte, 2>` — 2 字节响应：`[0x00][0x00]`。

### 调用（向下）

无

### 被调用（向上）

- [[protocol/vless/relay|relay]] `handshake()` 发送响应

### 知识域

VLESS 响应格式、客户端兼容性

---

## 函数: build_udp_packet()

### 功能说明

构建 VLESS UDP 数据包。格式为 `[ATYP(1)][ADDR(var)][PORT(2)][Payload]`，与 Trojan 不同，不含 Length 和 CRLF 字段。ATYP 值与 VLESS 请求头一致：IPv4=0x01, Domain=0x02, IPv6=0x03。

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

- [[protocol/vless/relay|relay]] `udp_frame_loop()` 构建响应

### 知识域

VLESS UDP 帧格式

---

## 函数: parse_udp_packet()

### 功能说明

从 TLS 流中读取的 UDP 数据包中解析出目标地址、端口和 payload。最小有效帧长度为 7 字节（ATYP + IPv4 + PORT）。

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

- [[protocol/vless/relay|relay]] `udp_frame_loop()` 解析入站数据

### 知识域

VLESS UDP 帧格式

## 相关页面

- [[protocol/vless/constants|constants]] — 协议常量
- [[protocol/vless/relay|relay]] — 协议中继器
- [[protocol/vless/message|message]] — 消息结构
