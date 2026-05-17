---
layer: core
source: "I:/code/Prism/include/prism/protocol/vless/format.hpp"
---

# VLESS 协议格式编解码

> 源码位置: `I:/code/Prism/include/prism/protocol/vless/format.hpp`

## 概述

VLESS 协议格式编解码模块，提供 VLESS 协议报文的底层解析函数声明，包括请求解析和响应序列化。

## 命名空间

```cpp
namespace psm::protocol::vless::format
```

## 核心结构

### udp_frame

VLESS UDP 帧信息结构，描述一个 VLESS UDP 数据包的目标地址和端口。

```cpp
struct udp_frame
{
    address destination_address;    // 目标地址
    std::uint16_t destination_port; // 目标端口
};
```

### udp_parse_result

VLESS UDP 数据包解析结果结构。

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

### parse_request

从 wire buffer 解析 VLESS 请求头。

```cpp
auto parse_request(std::span<const std::uint8_t> buffer) -> std::optional<request>;
```

**请求格式**: `[Version 1B][UUID 16B][AddnlInfoLen 1B][AddnlInfo var][Cmd 1B][Port 2B BE][Atyp 1B][Addr var]`

**限制**: 仅支持 plain VLESS（AddnlInfoLen = 0）

**返回**: 解析成功返回请求结构，失败返回 `std::nullopt`

### make_response

获取 VLESS 响应字节数组。

```cpp
[[nodiscard]] constexpr auto make_response() -> std::array<std::byte, 2>
{
    return {static_cast<std::byte>(version), std::byte{0x00}};
}
```

**返回**: `[Version 0x00][Addons Length 0x00]` 的 2 字节数组

**重要**: 必须发送 2 字节，不能只发送 1 字节。客户端期望读取 2 字节响应，仅发送 1 字节会导致客户端将后续 smux ACK 数据误读为 Addons Length，造成流偏移。

### UDP 数据包编解码

```cpp
// 构建 VLESS UDP 数据包
auto build_udp_packet(const udp_frame &frame, std::span<const std::byte> payload,
                      memory::vector<std::byte> &out) -> fault::code;

// 解析 VLESS UDP 数据包
auto parse_udp_packet(std::span<const std::byte> buffer)
    -> std::pair<fault::code, udp_parse_result>;
```

**UDP 帧格式**: `[ATYP(1)][ADDR(var)][PORT(2)][Payload]`

**与 Trojan 的区别**: VLESS UDP 不含 Length 和 CRLF 字段

**ATYP 值**: IPv4=0x01, Domain=0x02, IPv6=0x03（与 VLESS 请求头一致）

## 调用链

- [[core/protocol/vless/constants|Constants]] - 版本号和常量定义
- [[core/protocol/vless/relay|Relay]] - 中继器使用这些解析函数
- [[core/protocol/vless/config|Config]] - 配置传递给中继器
- [[core/fault/code|Fault Code]] - 错误码处理