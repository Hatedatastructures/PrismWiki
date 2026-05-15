---
title: "transparent.hpp — 透明哈希与相等比较器"
source: "include/prism/resolve/dns/detail/transparent.hpp"
module: "resolve"
type: api
tags: [resolve, dns, transparent, 哈希]
created: 2026-05-15
updated: 2026-05-15
---

# transparent.hpp

> 源码: `include/prism/resolve/dns/detail/transparent.hpp`
> 模块: [[resolve|Resolve]] / dns / detail

## 概述

提供支持异构查找的哈希函数和相等比较器，允许在 `unordered_map` 中使用 `string_view` 和 `memory::string` 进行混合查找，无需构造临时键对象。采用 FNV-1a 算法实现高效哈希。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | memory::string 类型 |
| 被依赖 | [[resolve/dns/detail/cache|cache]] | 缓存使用透明哈希 |
| 被依赖 | [[resolve/dns/detail/coalescer|coalescer]] | 合并器使用透明哈希 |

## 命名空间

`psm::resolve::dns::detail`

---

## 结构体: transparent_hash

### 概述
透明字符串哈希函数对象。支持对 `std::string_view` 和 `memory::string` 进行哈希计算。采用 FNV-1a 算法，使用 64 位种子和质数常数进行逐字节哈希运算。

### 设计意图
通过 `is_transparent` 类型别名启用异构查找特性，避免查找时构造临时 `memory::string` 对象。

### 成员函数

#### `operator()(string_view)`
```cpp
[[nodiscard]] auto operator()(const std::string_view value) const noexcept -> std::size_t;
```
计算 `string_view` 的哈希值。

#### `operator()(memory::string)`
```cpp
[[nodiscard]] auto operator()(const memory::string &value) const noexcept -> std::size_t;
```
计算 `memory::string` 的哈希值，转换为 `string_view` 后计算。

---

## 结构体: transparent_equal

### 概述
透明字符串相等比较函数对象。支持对 `std::string_view` 和 `memory::string` 进行混合比较。

### 成员函数

提供四种 `operator()` 重载覆盖所有类型组合：
- `(string_view, string_view)`
- `(memory::string, string_view)`
- `(string_view, memory::string)`
- `(memory::string, memory::string)`

---

## 知识域

- [[ref/programming/constexpr|异构查找]]
- [[ref/crypto/blake3|FNV-1a 哈希算法]]
