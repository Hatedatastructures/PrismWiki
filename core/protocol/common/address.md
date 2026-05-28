---
layer: core
source: I:/code/Prism/include/prism/protocol/common/address.hpp
title: 共享地址类型
tags: [protocol, common, address, ipv4, ipv6, domain, variant]
---

# 共享地址类型

定义跨协议通用的地址结构，包括 IPv4、IPv6 和域名三种类型。

## 源码位置

`I:/code/Prism/include/prism/protocol/common/address.hpp`

## 类型定义

### ipv4_address

IPv4 地址结构，4 字节网络字节序。

```cpp
struct ipv4_address
{
    std::array<std::uint8_t, 4> bytes;
};
```

### ipv6_address

IPv6 地址结构，16 字节网络字节序。

```cpp
struct ipv6_address
{
    std::array<std::uint8_t, 16> bytes;
};
```

### domain_address

域名地址结构，长度前缀编码格式。

```cpp
struct domain_address
{
    std::uint8_t length;                  // 1-255
    std::array<char, 255> value;          // 域名内容
    auto to_string(memory::resource_pointer mr) -> memory::string;
};
```

### address

通用地址变体类型，类型安全的多态封装。

```cpp
using address = std::variant<ipv4_address, ipv6_address, domain_address>;
```

## 工具函数

### address_to_string

将地址变体转换为可读字符串，支持自定义内存分配器。

```cpp
auto address_to_string(const address &addr, memory::resource_pointer mr)
    -> memory::string;
```

- IPv4/IPv6 使用 `inet_ntop` 格式化
- 域名直接返回原始内容

## 设计特点

- **POD 类型**：可直接从协议缓冲区拷贝填充
- **零拷贝友好**：域名内容以 `string_view` 形式返回
- **PMR 支持**：支持自定义内存分配器

## 使用协议

以下协议通过 `using` 声明引用这些共享类型：

- [[core/protocol/socks5/wire]] - SOCKS5 线级解析
- [[core/protocol/socks5/stream]] - SOCKS5 中继器

## 调用链

```
protocol/socks5/wire::parse_ipv4 -> ipv4_address
protocol/socks5/wire::parse_domain -> domain_address
protocol/common::address_to_string -> memory::string
```