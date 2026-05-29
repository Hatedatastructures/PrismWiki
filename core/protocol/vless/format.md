---
layer: core
source: "include/prism/protocol/vless/framing.hpp"
title: VLESS 协议格式编解码
tags: [protocol, vless, format, framing, uuid, parse, udp]
---

# VLESS 协议格式编解码

> 源码位置: `include/prism/protocol/vless/framing.hpp`

## 概述

VLESS 协议格式编解码模块，提供请求解析和响应序列化。实现位于 `src/prism/protocol/vless/framing.cpp`。地址/端口解析委托给 `common::framing`。

## 命名空间

```cpp
namespace psm::protocol::vless::format
```

## VLESS 请求字节布局

```
VLESS 请求:
┌─────────┬──────────┬──────────────┬─────────────┬──────┬──────────┬──────┬──────────┐
│ Version │ UUID     │ AddnlInfoLen │ AddnlInfo   │ Cmd  │ Port     │ Atyp │ Addr     │
│ 1B      │ 16B      │ 1B           │ 变长        │ 1B   │ 2B BE    │ 1B   │ 变长     │
│ =0x00   │ 二进制   │ =0(plain)    │ (无)        │      │          │      │          │
└─────────┴──────────┴──────────────┴─────────────┴──────┴──────────┴──────┴──────────┘
最小长度: 1 + 16 + 1 + 1 + 2 + 1 + 4 = 26 字节 (IPv4)

UUID 二进制表示:
  字符串 "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" 转为 16 字节二进制
  解析时直接 memcpy 16 字节，不做格式转换

Cmd 值: 0x01=TCP, 0x02=UDP, 0x7F=MUX
Atyp 值: 0x01=IPv4, 0x02=Domain, 0x03=IPv6 (注意: 与 Trojan/SOCKS5 不同!)
```

```
VLESS 响应:
┌─────────┬───────────────┐
│ Version │ Addons Length │
│ 1B      │ 1B            │
│ =0x00   │ =0x00         │
└─────────┴───────────────┘
固定 2 字节
```

```
VLESS UDP 帧:
┌──────┬──────────────────┬──────────┬──────────┐
│ ATYP │ ADDR + PORT      │ Payload  │          │
│ 1B   │ 变长             │ 变长     │          │
└──────┴──────────────────┴──────────┴──────────┘
最小长度: ATYP(1) + IPv4(4) + PORT(2) = 7 字节
注意: 无 Length 和 CRLF 字段（与 Trojan 不同）
Payload = 地址端口之后的全部数据
```

## 核心函数

### parse_request

```cpp
auto parse_request(std::span<const std::uint8_t> buffer) -> std::optional<request>;
```

**解析流程**:
1. 检查最小长度（26 字节）
2. 验证版本号（必须 0x00）
3. memcpy 16 字节 UUID
4. 验证 AddnlInfoLen（必须 0，否则返回 nullopt）
5. 解析命令 → 映射到 form（tcp/mux→stream, udp→datagram）
6. 解析端口（委托 common::framing::parse_port）
7. 解析地址（委托 common::framing::parse_*）

> **约束**: XTLS/Vision flow 不支持。`addnl_info_len != 0` 时直接返回 nullopt。客户端配置 Vision flow 时连接立即失败。

### make_response

```cpp
[[nodiscard]] constexpr auto make_response() -> std::array<std::byte, 2>
{
    return {static_cast<std::byte>(version), std::byte{0x00}};
}
```

> **约束**: 必须发送 2 字节，不能只发送 1 字节。客户端期望读取 2 字节响应，仅发送 1 字节会导致客户端将后续 smux ACK 数据误读为 Addons Length，造成流偏移。

### UDP 编解码

```cpp
auto build_udp_pkt(const udp_routed &frame, std::span<const std::byte> payload,
                   memory::vector<std::byte> &out) -> fault::code;

auto parse_udp_pkt(std::span<const std::byte> buffer)
    -> std::pair<fault::code, udp_parse_result>;
```

**与 Trojan UDP 的区别**: VLESS UDP 帧不含 Length 和 CRLF 字段。Payload 为地址端口之后的全部数据。

## 实现边界

- **固定 320 字节缓冲区**: 最大请求 278 字节（domain=255），320 字节够用但无余量
- **UUID 验证失败无额外资源分配**: 此时仅读取了固定缓冲区，不存在泄漏

详见 [[dev/debugging/deep-dive/protocol-boundaries|代理协议实现边界与认证深层分析]]

## 调用链

- [[core/protocol/vless/constants|Constants]] - 版本号和常量定义
- [[core/protocol/common/framing|Framing]] - 共享地址/端口解析
- [[core/protocol/vless/relay|Relay]] - 中继器使用这些解析函数
- [[core/fault/code|Fault Code]] - 错误码处理
