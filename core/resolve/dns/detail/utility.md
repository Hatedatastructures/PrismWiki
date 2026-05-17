---
title: "utility — DNS 解析器工具函数"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/detail/utility.hpp"
module: "resolve/dns/detail"
type: utility
tags: [dns, utility, zero-alloc, parse, port]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/router
---

# utility — DNS 解析器工具函数

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/detail/utility.hpp`
> 模块: [[resolve|resolve]] / [[resolve/dns|dns]] / detail
> 类型: header-only 工具函数

## 组件定位

该文件提供 DNS 解析器模块使用的零分配工具函数，避免 `string_view` 转整数时的中间堆分配。所有函数均为 inline header-only 实现。

## parse_port 函数

```cpp
[[nodiscard]] inline auto parse_port(const std::string_view port_str) noexcept
    -> std::optional<std::uint16_t>
{
    if (port_str.empty() || port_str.size() > 5) {
        return std::nullopt;
    }

    std::uint32_t value = 0;
    const auto result = std::from_chars(
        port_str.data(), port_str.data() + port_str.size(), value);

    if (result.ec != std::errc{} || value > 65535) {
        return std::nullopt;
    }

    return static_cast<std::uint16_t>(value);
}
```

### 实现要点

1. **长度校验**: 端口字符串长度必须在 1-5 之间（`"0"` 到 `"65535"`）
2. **零分配解析**: 使用 `std::from_chars` 直接从 `string_view` 解析
3. **范围校验**: 检查解析值是否在 `0-65535` 范围内
4. **错误处理**: 失败返回 `std::nullopt`

### 传统方式对比

```cpp
// 传统方式（有堆分配）
auto port = std::stoi(std::string(port_str));  // 构造临时 string

// 零分配方式（本项目）
auto port = parse_port(port_str);  // 无临时对象
```

## 使用场景

在 [[core/resolve/router|router]] 中用于端口解析：

```cpp
auto router::async_forward(std::string_view host, std::string_view port)
    -> net::awaitable<std::pair<fault::code, pooled_connection>>
{
    // IP 字面量检测
    const auto addr = net::ip::make_address(host, ec);
    if (!ec) {
        const auto port_num = parse_port(port).value_or(0);  // 零分配端口解析
        const tcp::endpoint ep(addr, port_num);
        // ...
    }
}
```

**优势**: 在路由热路径中避免堆分配，保持性能。

## 参见

- [[core/resolve/router|router]] — 分发层路由器（使用 parse_port）