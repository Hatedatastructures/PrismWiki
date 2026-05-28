---
title: "transparent -- 透明哈希与相等比较器"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/detail/transparent.hpp"
module: "resolve/dns/detail"
type: utility
tags: [dns, hash, transparent, heterogeneous-lookup, fnv-1a]
created: 2026-05-17
updated: 2026-05-28
related:
  - core/resolve/dns/detail/cache
  - core/resolve/dns/detail/coalescer
  - core/resolve/router
---

# transparent -- 透明哈希与相等比较器

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/detail/transparent.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]] / detail

## 组件定位

该文件提供支持异构查找的哈希函数和相等比较器，允许在 `unordered_map` 中使用 `string_view` 和 `memory::string` 进行混合查找，无需构造临时键对象。

## transparent_hash 结构

**算法**: FNV-1a（64 位），种子 `14695981039346656037ull`，质数 `1099511628211ull`。

**特点**: 逐字节混合，高位快速传播，分布均匀，无内存访问，适合小字符串。

**异构查找支持**: 定义 `using is_transparent = void;`，C++ 标准库通过此类型别名检测异构查找支持。

支持的重载：
- `operator()(string_view) -> size_t` -- 委托给内部 `hash_view`
- `operator()(memory::string) -> size_t` -- 转为 `string_view` 后委托

## transparent_equal 结构

同样定义 `using is_transparent = void;`，提供四种 `operator()` 重载覆盖所有类型组合：
- `(string_view, string_view)`
- `(memory::string, string_view)`
- `(string_view, memory::string)`
- `(memory::string, memory::string)`

所有重载内部转为 `string_view` 比较委托。

## 性能优势

| 操作 | 传统方式 | 透明哈希 |
|------|----------|----------|
| `map.find("key")` | 构造临时 `string` | 直接使用 `string_view` |
| 堆分配 | 1 次 | 0 次 |
| 内存拷贝 | 字符串内容 | 无 |

**热路径意义**: DNS 解析和路由查找是高频操作，透明哈希避免了每次查找的临时对象构造开销。

## 使用位置

| 消费者 | 用途 |
|--------|------|
| [[core/resolve/dns/detail/cache|cache]] | 缓存键 `"domain:qtype"` 的零分配查找 |
| [[core/resolve/dns/detail/coalescer|coalescer]] | 请求合并索引的 `string_view` 键查找 |
| [[core/resolve/router|router]] | 反向路由表的异构查找 |

## 参见

- [[core/resolve/router|string_hash]] -- router 中使用的透明哈希
- [[core/resolve/dns/detail/cache|cache]] -- DNS 缓存
- [[core/resolve/dns/detail/coalescer|coalescer]] -- 请求合并器
