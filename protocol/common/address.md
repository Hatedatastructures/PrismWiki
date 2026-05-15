---
title: "address — 共享地址类型定义"
source: "include/prism/protocol/common/address.hpp"
module: "protocol"
type: api
tags: [protocol, common, address, IPv4, IPv6, domain, variant, PMR]
related:
  - "[[protocol/socks5/message|socks5_message]]"
  - "[[protocol/trojan/message|trojan_message]]"
  - "[[protocol/vless/message|vless_message]]"
  - "[[protocol/shadowsocks/message|ss_message]]"
  - "[[memory|memory]]"
created: 2026-05-15
updated: 2026-05-15
---

# address.hpp

> 源码: `include/prism/protocol/common/address.hpp`
> 模块: [[protocol|protocol]] > common

## 概述

共享地址类型定义。定义跨协议通用的地址结构，包括 IPv4、IPv6 和域名三种类型。各协议（SOCKS5/Trojan/VLESS/Shadowsocks）通过 `using` 声明引用这些共享类型，消除四个 `message.hpp` 中的重复定义。地址使用 `std::variant` 实现类型安全的多态，所有结构设计为零拷贝友好。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory|memory]] | `memory::string`、`memory::resource_pointer` |
| 被依赖 | [[protocol/socks5/message|socks5_message]] | 通过 `using` 引用地址类型 |
| 被依赖 | [[protocol/trojan/message|trojan_message]] | 通过 `using` 引用地址类型 |
| 被依赖 | [[protocol/vless/message|vless_message]] | 通过 `using` 引用地址类型 |
| 被依赖 | [[protocol/shadowsocks/message|ss_message]] | 通过 `using` 引用地址类型 |

## 命名空间

`psm::protocol::common`

---

## 结构体: ipv4_address

IPv4 地址结构。包含 4 字节的 IPv4 地址数据，采用网络字节序存储。POD 类型，可直接从协议缓冲区拷贝填充。

| 字段 | 类型 | 说明 |
|------|------|------|
| `bytes` | `std::array<std::uint8_t, 4>` | 地址字节数组（网络字节序） |

---

## 结构体: ipv6_address

IPv6 地址结构。包含 16 字节的 IPv6 地址数据，采用网络字节序存储。

| 字段 | 类型 | 说明 |
|------|------|------|
| `bytes` | `std::array<std::uint8_t, 16>` | 地址字节数组（网络字节序） |

---

## 结构体: domain_address

域名地址结构。包含域名长度和内容，遵循代理协议的域名编码格式。域名最大长度为 255 字节（1 字节长度字段限制）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `length` | `std::uint8_t` | 域名长度（1-255） |
| `value` | `std::array<char, 255>` | 域名内容缓冲区 |

### 函数: to_string()

#### 功能说明

根据指定的内存资源创建域名字符串。

#### 签名

```cpp
[[nodiscard]] auto to_string(const memory::resource_pointer mr = memory::current_resource()) const
    -> memory::string;
```

#### 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `mr` | `memory::resource_pointer` | `memory::current_resource()` | 内存资源指针 |

#### 返回值

`memory::string` — 域名字符串。

#### 调用（向下）

- `memory::string` 构造函数

#### 被调用（向上）

- `address_to_string()` 转换域名地址

#### 知识域

- [[protocol/common/address|address]] 地址类型、域名编码格式

---

## 类型别名: address

```cpp
using address = std::variant<ipv4_address, ipv6_address, domain_address>;
```

通用地址变体类型。使用 `std::variant` 封装三种地址类型，提供类型安全的多态访问。访问者模式配合 `std::visit` 可实现编译期类型分发。

---

## 函数: address_to_string()

### 功能说明

将地址变体转换为可读的字符串表示。IPv4 和 IPv6 地址使用 `inet_ntop` 进行格式化，域名直接返回原始内容。支持自定义内存分配器。

### 签名

```cpp
[[nodiscard]] inline auto address_to_string(const address &addr,
                                             memory::resource_pointer mr = memory::current_resource())
    -> memory::string;
```

### 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `addr` | `const address &` | — | 地址变体 |
| `mr` | `memory::resource_pointer` | `memory::current_resource()` | 内存资源指针 |

### 返回值

`memory::string` — 地址字符串。

### 调用（向下）

- `inet_ntop()` — IPv4/IPv6 地址格式化
- `domain_address::to_string()` — 域名转换

### 被调用（向上）

- 各协议的 `to_string()` 函数
- 日志输出

### 知识域

`inet_ntop` API、`std::visit` 访问者模式

## 相关页面

- [[protocol/socks5/message|socks5_message]] — SOCKS5 消息结构
- [[protocol/trojan/message|trojan_message]] — Trojan 消息结构
- [[protocol/vless/message|vless_message]] — VLESS 消息结构
- [[memory|memory]] — PMR 内存管理
