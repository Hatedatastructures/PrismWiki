---
title: "frame — smux 帧协议定义"
source: "include/prism/multiplex/smux/frame.hpp"
implementation: "src/prism/multiplex/smux/frame.cpp"
module: "multiplex"
type: api
tags: [multiplex, smux, frame, 帧格式, 编解码]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/smux/craft|smux::craft]]"
  - "[[multiplex/smux/config|smux::config]]"
  - "[[multiplex/parcel|parcel]]"
  - "[[memory/container|container]]"
---

# frame.hpp — smux 帧协议定义

> 源码: `include/prism/multiplex/smux/frame.hpp`
> 实现: `src/prism/multiplex/smux/frame.cpp`
> 模块: [[multiplex/smux/craft|multiplex]] > smux

## 概述

smux 帧协议定义。兼容 Mihomo/xtaci/smux v1，定义帧格式、命令类型和编解码函数。

帧格式为 8 字节定长帧头：`[Version 1B][Cmd 1B][Length 2B LE][StreamID 4B LE]`。Length 和 StreamID 采用小端字节序。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | PMR 容器（string, vector） |
| 被依赖 | [[multiplex/smux/craft|smux::craft]] | 帧编解码、地址解析 |
| 被依赖 | [[multiplex/parcel|parcel]] | UDP 数据报解析/构建 |
| 被依赖 | [[multiplex/yamux/craft|yamux::craft]] | 复用 parse_mux_address 解析地址 |

## 命名空间

`psm::multiplex::smux`

---

## 常量

| 常量 | 值 | 说明 |
|------|----|------|
| `protocol_version` | `0x01` | 协议版本号 |
| `frame_header_size` | `8` | 帧头大小（字节） |
| `max_frame_length` | `65535` | 最大帧数据大小 |

---

## 枚举: command

**功能说明**

smux 帧命令类型。命令值对齐 Mihomo 使用的 xtaci/smux 协议。

**签名**

```cpp
enum class command : std::uint8_t {
    syn  = 0,  // 新建流
    fin  = 1,  // 半关闭流
    push = 2,  // 数据推送
    nop  = 3,  // 心跳（不回复）
};
```

**参数**

无（枚举类型）。

**返回值**

枚举值，用于帧头的 Cmd 字段。

**调用（向下）**

无。

**被调用（向上）**

- `smux::craft::frame_loop()` — 根据命令类型分发
- `smux::craft::push_frame()` — 编码帧头时设置命令

**知识域**

smux 协议命令定义、xtaci/smux 兼容性。

---

## 结构体: frame_header

**功能说明**

smux 帧头结构，8 字节定长。定长帧头格式为 `[Version 1B][Cmd 1B][Length 2B LE][StreamID 4B LE]`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | `uint8_t` | 协议版本号 |
| `cmd` | `command` | 命令类型 |
| `length` | `uint16_t` | 负载长度（小端序） |
| `stream_id` | `uint32_t` | 流标识符（小端序） |

---

## 结构体: parsed_address

**功能说明**

从 mux 首个 PSH 帧解析出的目标地址。sing-mux StreamRequest 格式：`[Flags 2B][ATYP 1B][Addr(var)][Port 2B]`。Flags bit0 标识 UDP 流，bit1 标识 PacketAddr 模式。支持 IPv4 (ATYP=0x01)、域名 (ATYP=0x03)、IPv6 (ATYP=0x04)。

| 字段 | 类型 | 说明 |
|------|------|------|
| `host` | `memory::string` | 目标主机 |
| `port` | `uint16_t` | 目标端口 |
| `offset` | `size_t` | 地址结束位置 |
| `is_udp` | `bool` | Flags bit0 |
| `packet_addr` | `bool` | Flags bit1 |

---

## 结构体: udp_datagram

**功能说明**

UDP 数据报解析结果。格式：`[ATYP 1B][Addr(var)][Port 2B][Length 2B BE][Payload]`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `host` | `memory::string` | 目标主机 |
| `port` | `uint16_t` | 目标端口 |
| `payload` | `span<const byte>` | 数据部分 |
| `consumed` | `size_t` | 解析消耗的总字节数 |

---

## 结构体: udp_length_prefixed

**功能说明**

Length-prefixed UDP 数据报解析结果。sing-mux 无 PacketAddr 模式格式：`[Length 2B BE][Payload]`。

| 字段 | 类型 | 说明 |
|------|------|------|
| `payload` | `span<const byte>` | 数据部分 |
| `consumed` | `size_t` | 解析消耗的总字节数 |

---

## 函数: deserialization

**功能说明**

解析帧头。从字节序列中解析 8 字节 smux 帧头，校验版本号和命令类型有效性。

**签名**

```cpp
[[nodiscard]] auto deserialization(std::span<const std::byte> data)
    -> std::optional<frame_header>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `span<const byte>` | 包含帧头的字节序列（至少 8 字节） |

**返回值**

`std::optional<frame_header>` — 解析成功的帧头，或 nullopt（校验失败）。

**调用（向下）**

无。

**被调用（向上）**

- `smux::craft::frame_loop()` — 每帧读取后解析帧头

**知识域**

小端序解析、帧头校验、版本号/命令类型验证。

---

## 函数: parse_mux_address

**功能说明**

解析 mux 首个 PSH 中的 Flags+地址。sing-mux StreamRequest 格式：`[Flags 2B][ATYP 1B][Addr][Port 2B]`。支持 IPv4/域名/IPv6 三种地址类型。

**签名**

```cpp
[[nodiscard]] auto parse_mux_address(std::span<const std::byte> data, memory::resource_pointer mr)
    -> std::optional<parsed_address>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `span<const byte>` | 包含地址的字节序列 |
| `mr` | `resource_pointer` | 内存资源 |

**返回值**

`std::optional<parsed_address>` — 解析结果，nullopt 表示数据不足或格式错误。

**调用（向下）**

- `inet_ntop()` — IPv6 地址格式化
- 内部 `format_ipv4()` — IPv4 地址格式化（手写，避免 printf 开销）

**被调用（向上）**

- `smux::craft::activate_stream()` — 解析目标地址
- `yamux::craft::activate_stream()` — 复用 smux 地址解析

**知识域**

SOCKS5 地址格式、ATYP 类型、sing-mux Flags 语义。

---

## 函数: parse_udp_datagram

**功能说明**

解析 UDP 数据报（SOCKS5 地址格式）。格式：`[ATYP 1B][Addr(var)][Port 2B][Length 2B BE][Payload]`。

**签名**

```cpp
[[nodiscard]] auto parse_udp_datagram(std::span<const std::byte> data, memory::resource_pointer mr)
    -> std::optional<udp_datagram>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `span<const byte>` | 包含数据报的字节序列 |
| `mr` | `resource_pointer` | 内存资源 |

**返回值**

`std::optional<udp_datagram>` — 解析结果，nullopt 表示数据不足或格式错误。

**调用（向下）**

- `inet_ntop()` — IPv6 地址格式化
- 内部 `format_ipv4()` — IPv4 地址格式化

**被调用（向上）**

- `parcel::process_buffer()` — PacketAddr 模式下解析 UDP 数据报

**知识域**

SOCKS5 UDP relay 格式、PacketAddr 模式。

---

## 函数: parse_udp_length_prefixed

**功能说明**

解析 length-prefixed UDP 数据报。格式：`[Length 2B BE][Payload]`。目标地址在 SYN 时已确定，不包含在数据帧中。

**签名**

```cpp
[[nodiscard]] auto parse_udp_length_prefixed(std::span<const std::byte> data)
    -> std::optional<udp_length_prefixed>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `span<const byte>` | 包含数据报的字节序列 |

**返回值**

`std::optional<udp_length_prefixed>` — 解析结果，nullopt 表示数据不足。

**调用（向下）**

无。

**被调用（向上）**

- `parcel::process_buffer()` — 非 PacketAddr 模式下解析 UDP 数据报

**知识域**

length-prefixed 编码、大端序解析。

---

## 函数: build_udp_datagram

**功能说明**

构建 UDP 数据报。格式：`[ATYP][addr][port][Length 2B BE][Payload]`。支持 IPv4/域名/IPv6 三种地址类型。

**签名**

```cpp
[[nodiscard]] auto build_udp_datagram(std::string_view host, std::uint16_t port,
                                      std::span<const std::byte> payload, memory::resource_pointer mr)
    -> memory::vector<std::byte>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `host` | `string_view` | 目标主机 |
| `port` | `uint16_t` | 目标端口 |
| `payload` | `span<const byte>` | 数据负载 |
| `mr` | `resource_pointer` | 内存资源 |

**返回值**

`memory::vector<std::byte>` — 编码后的完整 UDP 数据报。

**调用（向下）**

- `inet_pton()` — 地址类型判断和转换

**被调用（向上）**

- `parcel::downlink_loop()` — PacketAddr 模式下编码 UDP 响应

**知识域**

SOCKS5 地址编码、sing-mux PacketAddr 格式。

---

## 函数: build_udp_length_prefixed

**功能说明**

构建 length-prefixed UDP 数据报（响应格式）。格式：`[Length 2B BE][Payload]`。

**签名**

```cpp
[[nodiscard]] auto build_udp_length_prefixed(std::span<const std::byte> payload, memory::resource_pointer mr)
    -> memory::vector<std::byte>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `payload` | `span<const byte>` | 数据负载 |
| `mr` | `resource_pointer` | 内存资源 |

**返回值**

`memory::vector<std::byte>` — 编码后的 `[Length 2B BE][Payload]`。

**调用（向下）**

无。

**被调用（向上）**

- `parcel::downlink_loop()` — 非 PacketAddr 模式下编码 UDP 响应

**知识域**

length-prefixed 编码、大端序写入。
