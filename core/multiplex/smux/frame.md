---
layer: core
source: I:/code/Prism/include/prism/multiplex/smux/frame.hpp
title: smux::frame - smux 帧协议定义
---

# smux::frame - smux 帧协议定义

## 源码位置

`I:/code/Prism/include/prism/multiplex/smux/frame.hpp`

## 概述

定义 smux 多路复用协议的帧格式、命令类型和编解码函数。兼容 Mihomo/xtaci/smux v1。

## 协议常量

```cpp
constexpr std::uint8_t protocol_version = 0x01;
constexpr std::size_t frame_header_size = 8;
constexpr std::size_t max_frame_length = 65535;
```

## 帧头结构

```cpp
struct frame_header
{
    std::uint8_t version = protocol_version;  // 协议版本号
    command cmd = command::push;              // 命令类型
    std::uint16_t length = 0;                 // 负载长度（小端序）
    std::uint32_t stream_id = 0;              // 流标识符（小端序）
};
```

帧格式：`[Version 1B][Cmd 1B][Length 2B LE][StreamID 4B LE]`

## 命令类型

```cpp
enum class command : std::uint8_t
{
    syn = 0,   // 新建流
    fin = 1,   // 半关闭流
    push = 2,  // 数据推送
    nop = 3    // 心跳（不回复）
};
```

## 地址解析结构

### parsed_address

从 mux 首个 PSH 帧解析的目标地址：

```cpp
struct parsed_address
{
    memory::string host;      // 目标主机
    std::uint16_t port = 0;   // 目标端口
    std::size_t offset = 0;   // 地址结束位置
    bool is_udp = false;      // 是否为 UDP 流
    bool packet_addr = false; // 是否为 PacketAddr 模式
};
```

sing-mux StreamRequest 格式：`[Flags 2B][ATYP 1B][Addr(var)][Port 2B]`

Flags 含义：
- bit0：UDP 流标识
- bit1：PacketAddr 模式

### udp_datagram

UDP 数据报解析结果：

```cpp
struct udp_datagram
{
    memory::string host;                // 目标主机
    std::uint16_t port = 0;             // 目标端口
    std::span<const std::byte> payload; // 数据部分
    std::size_t consumed = 0;           // 解析消耗字节数
};
```

格式：`[ATYP 1B][Addr(var)][Port 2B][Data]`

### udp_length_prefixed

Length-prefixed UDP 数据报：

```cpp
struct udp_length_prefixed
{
    std::span<const std::byte> payload; // 数据部分
    std::size_t consumed = 0;           // 解析消耗字节数
};
```

格式：`[Length 2B BE][Payload]`

## 解析函数

```cpp
// 解析 UDP 数据报（SOCKS5 地址格式）
auto parse_udp_datagram(std::span<const std::byte> data,
                        memory::resource_pointer mr)
    -> std::optional<udp_datagram>;

// 解析 length-prefixed UDP 数据报
auto parse_udp_length_prefixed(std::span<const std::byte> data)
    -> std::optional<udp_length_prefixed>;

// 解析 mux 首个 PSH 中的 Flags+地址
auto parse_mux_address(std::span<const std::byte> data,
                       memory::resource_pointer mr)
    -> std::optional<parsed_address>;

// 解析帧头
auto deserialization(std::span<const std::byte> data)
    -> std::optional<frame_header>;
```

## 构建函数

```cpp
// 构建 UDP 数据报
auto build_udp_datagram(std::string_view host,
                        std::uint16_t port,
                        std::span<const std::byte> payload,
                        memory::resource_pointer mr)
    -> memory::vector<std::byte>;

// 构建 length-prefixed UDP 数据报
auto build_udp_length_prefixed(std::span<const std::byte> payload,
                               memory::resource_pointer mr)
    -> memory::vector<std::byte>;
```

## 地址类型支持

| ATYP | 地址类型 | 格式 |
|------|----------|------|
| 0x01 | IPv4 | 4 字节 |
| 0x03 | 域名 | [Length 1B][Name] |
| 0x04 | IPv6 | 16 字节 |

## 关联文档

- [[core/multiplex/smux/craft|smux::craft]] - smux 协议实现
- [[core/multiplex/parcel|parcel]] - UDP 数据报管道（使用解析函数）