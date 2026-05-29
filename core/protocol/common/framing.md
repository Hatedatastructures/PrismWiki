---
title: 共享协议帧解析
created: 2026-05-25
updated: 2026-05-27
layer: core
source: include/prism/protocol/common/framing.hpp
tags: [protocol, common, framing, parse, ipv4, ipv6, domain, port]
---

# 共享协议帧解析

提供跨协议通用的地址和端口线级解析函数。各协议（SOCKS5/Trojan/VLESS/Shadowsocks）通过 `using` 声明复用这些函数，消除四处重复的 memcpy 实现。

> 源码: include/prism/protocol/common/framing.hpp | 实现: inline header-only

## 命名空间

`psm::protocol::common::framing`

## 函数列表

### parse_ipv4

```cpp
[[nodiscard]] inline auto parse_ipv4(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv4_address>;
```

解析 4 字节 IPv4 地址（至少 4 字节）。

### parse_ipv6

```cpp
[[nodiscard]] inline auto parse_ipv6(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, ipv6_address>;
```

解析 16 字节 IPv6 地址（至少 16 字节）。

### parse_domain

```cpp
[[nodiscard]] inline auto parse_domain(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, domain_address>;
```

解析域名地址（LEN(1) + DOMAIN(n)）。长度字段 1 字节，域名最长 255 字节。

### parse_port

```cpp
[[nodiscard]] inline auto parse_port(std::span<const std::uint8_t> buffer)
    -> std::pair<fault::code, std::uint16_t>;
```

解析大端序端口号（至少 2 字节）。

## 委托关系

```
common::framing (parse_ipv4/parse_ipv6/parse_domain/parse_port)
  ├── socks5::wire      (using 委托)
  ├── trojan::format    (using 委托)
  └── shadowsocks::format (独立调用)
```

> **约束**: 所有使用 `using` 委托的协议，其地址解析行为完全等价。修改 `common::framing` 的任何解析逻辑将自动影响所有委托协议。若某协议需要特殊解析行为（如 Shadowsocks），必须自行实现，不受 using 委托约束。

## 设计决策

### 为什么用委托模式（using 声明）而不是继承或模板？

SOCKS5、Trojan、VLESS 等协议的地址解析字节格式完全相同（IPv4 4 字节、IPv6 16 字节、域名 LEN+VALUE、端口 2 字节大端序），但各协议命名空间希望暴露自己的一套函数签名（如 `socks5::wire::parse_ipv4`）。

使用 `using` 声明将 `common::framing` 的函数导入各协议命名空间，既能保持命名空间隔离（调用方写 `socks5::wire::parse_ipv4`），又避免任何虚函数开销或模板实例化膨胀。这是零成本抽象的直接体现。

### 为什么 Shadowsocks 不用 using 委托？

Shadowsocks SS2022 的地址格式虽然与 SOCKS5 相同（ATYP+ADDR+PORT），但其 `format` 命名空间提供了额外的解析函数（如 `parse_addr_port` 将地址和端口一起解析）。独立的解析路径允许在解析后直接附加 SS2022 特有的时间戳验证和 AEAD 上下文操作。

## 使用场景

- **协议握手**: 每个代理协议的握手阶段读取客户端请求头后，调用对应的 `parse_*` 函数提取目标地址
- **UDP 帧解析**: Trojan/VLESS 的 UDP over TLS 实现解析帧内嵌地址时复用这些函数
- **日志输出**: 地址解析结果用于连接日志和调试追踪

## 设计特点

- 所有函数 `inline auto`，零开销
- 直接操作字节缓冲区，`memcpy` 实现
- 统一返回 `std::pair<fault::code, result>`
- 缓冲区不足时返回 `fault::code::bad_message`，不抛异常

## 依赖

- [[core/protocol/common/address|address]] - 地址类型定义
- [[core/fault/code|fault::code]] - 错误码
