---
layer: core
source: "I:/code/Prism/include/prism/protocol/trojan/format.hpp"
title: Trojan 协议格式编解码
---

# Trojan 协议格式编解码

> 源码位置: `I:/code/Prism/include/prism/protocol/trojan/format.hpp`

## 概述

Trojan 协议格式编解码模块，提供 Trojan 协议报文的底层解析函数声明，包括凭据解码、CRLF 验证、命令和地址类型解析、地址解析、端口解码以及 UDP 帧编解码。

## 命名空间

```cpp
namespace psm::protocol::trojan::format
```

## 核心结构

### header_parse

协议头部解析结果结构，存储从协议头部解析出的命令和地址类型。

```cpp
struct header_parse
{
    command cmd;       // 命令类型
    address_type atyp; // 地址类型
};
```

### udp_frame

Trojan UDP 帧信息结构，描述一个 Trojan UDP 数据包的目标地址和端口。

```cpp
struct udp_frame
{
    address destination_address;    // 目标地址
    std::uint16_t destination_port; // 目标端口
};
```

### udp_parse_result

Trojan UDP 数据包解析结果结构，包含解析出的目标地址、端口以及 payload 偏移和大小。

```cpp
struct udp_parse_result
{
    address destination_address;      // 目标地址
    std::uint16_t destination_port{}; // 目标端口
    std::size_t payload_offset{};     // payload 在缓冲区中的偏移
    std::size_t payload_size{};       // payload 大小
};
```

## 核心函数

### parse_credential

解析用户凭据，从缓冲区提取 56 字节的凭据数据。

```cpp
auto parse_credential(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, std::array<char, 56>>;
```

**参数**:
- `buffer`: 包含凭据的缓冲区，至少 56 字节

**返回**: 错误码和凭据数组

### parse_crlf

验证 CRLF 分隔符。

```cpp
auto parse_crlf(const std::span<const std::uint8_t> buffer)
    -> fault::code;
```

**参数**:
- `buffer`: 包含 CRLF 的缓冲区，至少 2 字节

**返回**: 验证结果错误码

### parse_cmd_atyp

解析命令和地址类型。

```cpp
auto parse_cmd_atyp(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, header_parse>;
```

**参数**:
- `buffer`: 包含命令和地址类型的缓冲区，至少 2 字节

**返回**: 错误码和解析结果

### 地址解析函数

```cpp
// 解析 IPv4 地址（至少 4 字节）
auto parse_ipv4(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv4_address>;

// 解析 IPv6 地址（至少 16 字节）
auto parse_ipv6(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv6_address>;

// 解析域名地址（长度字节加域名内容）
auto parse_domain(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, domain_address>;
```

### parse_port

解析端口号。

```cpp
auto parse_port(const std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, uint16_t>;
```

**参数**:
- `buffer`: 包含端口号的缓冲区，至少 2 字节

**返回**: 错误码和端口号

### UDP 数据包编解码

```cpp
// 构建 Trojan UDP 数据包（mihomo 兼容格式）
auto build_udp_packet(const udp_frame &frame, std::span<const std::byte> payload,
                      memory::vector<std::byte> &out)
    -> fault::code;

// 解析 Trojan UDP 数据包（mihomo 兼容格式）
auto parse_udp_packet(std::span<const std::byte> buffer)
    -> std::pair<fault::code, udp_parse_result>;
```

## 调用链

- [[core/protocol/trojan/constants|Constants]] - 命令和地址类型枚举定义
- [[core/protocol/trojan/relay|Relay]] - 协议中继器使用这些解析函数
- [[core/protocol/trojan/config|Config]] - 配置传递给中继器
- [[core/channel/transport/transmission|Transmission]] - 底层传输层接口

## 实现边界

- **56 字节 SHA224 明文传输**：密码以 hex 明文在 TLS 内层传输，TLS 配置错误（如降级）时密码暴露
- **缓冲区精确边界**：域地址长度 255 时 `required_total = 320`，恰好等于固定缓冲区大小，零余量
- **日志不区分错误类型**：`credential verification failed` 不区分"格式错误"还是"密码错误"

排障建议：`echo -n "password" | openssl dgst -sha224` 验证客户端密码配置

详见 [[dev/debugging/deep-dive/protocol-boundaries|代理协议实现边界与认证深层分析]]