---
layer: core
source: I:/code/Prism/include/prism/protocol/socks5/wire.hpp
title: SOCKS5 协议线级解析
tags: [protocol, socks5, wire, parse, udp, auth, header]
---

# SOCKS5 协议线级解析

提供 SOCKS5 协议报文的底层解析函数，包括头部、地址、端口和 UDP 数据报的编解码。

## 源码位置

`I:/code/Prism/include/prism/protocol/socks5/wire.hpp`

## 结构定义

### header_parse

SOCKS5 请求头部解析结果。

```cpp
struct header_parse
{
    std::uint8_t version;    // 固定 0x05
    command cmd;             // 命令类型
    std::uint8_t rsv;        // 保留字段
    address_type atyp;       // 地址类型
};
```

### udp_header

SOCKS5 UDP 数据报头部。

```cpp
struct udp_header
{
    address destination_address;
    std::uint16_t destination_port;
    std::uint8_t frag;       // 分片序号，0 表示独立数据报
};
```

### udp_header_parse

UDP 报头解析结果，包含头部大小用于定位 DATA 起始。

```cpp
struct udp_header_parse
{
    udp_header header;
    std::size_t header_size;
};
```

### password_auth_request

RFC 1929 用户名/密码认证请求。

```cpp
struct password_auth_request
{
    std::uint8_t version;      // 固定 0x01
    std::string_view username;
    std::string_view password;
};
```

## 解析函数

### parse_header

解析 4 字节请求头部（VER + CMD + RSV + ATYP）。

```cpp
auto parse_header(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, header_parse>;
```

### parse_ipv4 / parse_ipv6

解析 IP 地址。

```cpp
auto parse_ipv4(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv4_address>;
auto parse_ipv6(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv6_address>;
```

### parse_domain

解析域名地址（LEN + DOMAIN）。

```cpp
auto parse_domain(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, domain_address>;
```

### decode_port

解析 2 字节大端序端口。

```cpp
auto decode_port(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, uint16_t>;
```

### decode_udp_header

解码 SOCKS5 UDP 报头。

```cpp
auto decode_udp_header(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, udp_header_parse>;
```

**验证**：
- RSV 必须为 0x0000
- FRAG 必须为 0（不支持分片）

### parse_password_auth

解析 RFC 1929 认证请求（VER + ULEN + UNAME + PLEN + PASSWD）。

```cpp
auto parse_password_auth(std::span<const std::uint8_t> data)
    -> std::pair<fault::code, password_auth_request>;
```

## 编码函数

### encode_udp_header

编码 UDP 报头（不含 DATA）。

```cpp
auto encode_udp_header(const udp_header &header,
                       memory::vector<std::uint8_t> &out) -> fault::code;
```

### encode_udp_datagram

编码完整 UDP 数据报（报头 + DATA）。

```cpp
auto encode_udp_datagram(const udp_header &header,
                          std::span<const std::uint8_t> data,
                          memory::vector<std::uint8_t> &out) -> fault::code;
```

### build_password_auth_response

构建 RFC 1929 认证响应（VER + STATUS）。

```cpp
auto build_password_auth_response(bool success)
    -> std::array<std::uint8_t, 2>;
```

## UDP 数据报格式

```
RSV(2) + FRAG(1) + ATYP(1) + DST.ADDR(变长) + DST.PORT(2) + DATA(变长)
```

## 设计特点

- **零拷贝友好**：直接操作字节缓冲区
- **错误码统一**：使用 `fault::code` 系统
- **内存高效**：预分配最大缓冲区大小

## 调用链

```
protocol/socks5::relay::read_request_header -> wire::parse_header
protocol/socks5::relay::read_address -> wire::parse_ipv4/parse_ipv6
protocol/socks5::relay::read_domain_address -> wire::parse_domain
protocol/socks5::relay::relay_single_datagram -> wire::decode_udp_header
protocol/socks5::relay::relay_single_datagram -> wire::encode_udp_datagram
```

## 依赖

- [[core/protocol/socks5/constants]] - 协议常量
- [[core/protocol/common/address]] - 地址类型
- [[log/fault/code]] - 错误码系统