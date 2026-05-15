---
title: "utility.hpp — DNS 解析器工具函数"
source: "include/prism/resolve/dns/detail/utility.hpp"
module: "resolve"
type: api
tags: [resolve, dns, utility, 工具函数, 零分配]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/dns/dns
  - resolve/router
  - memory/container
---

# utility.hpp

> 源码: `include/prism/resolve/dns/detail/utility.hpp`
> 模块: [[resolve|Resolve]] / dns / detail

## 概述

提供 DNS 解析器模块使用的零分配工具函数，避免 `string_view` 转整数时的中间堆分配。所有函数均为 inline header-only 实现，使用 `std::from_chars` 进行高性能数值解析。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[resolve/dns/dns|dns]] | resolver_impl 使用 parse_port |
| 被依赖 | [[resolve/router|router]] | router 使用 parse_port 解析端口 |

## 命名空间

`psm::resolve::dns::detail`

---

## 函数: parse_port()

- **功能说明**: 从 `string_view` 解析端口号。使用 `std::from_chars` 零分配解析，不产生任何临时字符串或堆分配。先检查长度是否在 1-5 之间（端口号最大 65535 为 5 位数字），再进行数值转换和范围校验（0-65535）。
- **签名**:
  ```cpp
  [[nodiscard]] inline auto parse_port(const std::string_view port_str) noexcept
      -> std::optional<std::uint16_t>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `port_str` | `const std::string_view` | 端口字符串视图，如 `"443"`、`"8080"` |

- **返回值**: `std::optional<std::uint16_t>` — 解析成功的端口号（0-65535），失败返回 `std::nullopt`
- **调用（向下）**: `std::from_chars()` — 零分配数值解析
- **被调用（向上）**:
  - [[resolve/dns/dns|resolver_impl::resolve_tcp()]] 解析端口参数
  - [[resolve/dns/dns|resolver_impl::resolve_udp()]] 解析端口参数
  - [[resolve/router|router::async_forward()]] 解析 IP 字面量的端口
  - [[resolve/router|router::async_datagram()]] 解析 IP 字面量的端口
  - [[resolve/router|router::resolve_datagram_target()]] 解析 IP 字面量的端口
- **知识域**: [[ref/memory/pmr|零分配优化]]

---

## 实现细节

```cpp
[[nodiscard]] inline auto parse_port(const std::string_view port_str) noexcept
    -> std::optional<std::uint16_t>
{
    if (port_str.empty() || port_str.size() > 5)
    {
        return std::nullopt;
    }

    std::uint32_t value = 0;
    const auto result = std::from_chars(
        port_str.data(), port_str.data() + port_str.size(), value);

    if (result.ec != std::errc{} || value > 65535)
    {
        return std::nullopt;
    }

    return static_cast<std::uint16_t>(value);
}
```

关键设计决策：
- 使用 `std::uint32_t` 作为中间类型，避免 `from_chars` 对 `uint16_t` 的潜在截断问题
- `noexcept` 保证不抛异常，适合热路径调用
- `from_chars` 不依赖 locale，纯数值解析，性能最优

---

## 调用链总览

```
[[resolve/router|router::async_forward()]]   → parse_port(port) → 构造 endpoint
[[resolve/router|router::async_datagram()]]  → parse_port(port) → 构造 endpoint
[[resolve/dns/dns|resolver_impl::resolve_tcp()]] → parse_port(port) → 组合 TCP endpoint
[[resolve/dns/dns|resolver_impl::resolve_udp()]] → parse_port(port) → 组合 UDP endpoint
```

---

## 知识域

- [[ref/memory/pmr|零分配优化]]
- [[resolve/dns/dns|DNS 查询管道]]
- [[resolve/router|路由策略]]
