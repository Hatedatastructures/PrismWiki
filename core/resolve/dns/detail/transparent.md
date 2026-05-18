---
title: "transparent — 透明哈希与相等比较器"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/detail/transparent.hpp"
module: "resolve/dns/detail"
type: utility
tags: [dns, hash, transparent, heterogeneous-lookup, fnv-1a]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/detail/cache
  - core/resolve/dns/detail/coalescer
  - core/resolve/router
---

# transparent — 透明哈希与相等比较器

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/detail/transparent.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]] / detail

## 组件定位

该文件提供支持异构查找的哈希函数和相等比较器，允许在 `unordered_map` 中使用 `string_view` 和 `memory::string` 进行混合查找，无需构造临时键对象。

## transparent_hash 结构

```cpp
struct transparent_hash {
    using is_transparent = void;  // 启用异构查找标记

    auto operator()(const std::string_view value) const noexcept -> std::size_t;
    auto operator()(const memory::string &value) const noexcept -> std::size_t;
};
```

### FNV-1a 算法

```cpp
static constexpr std::size_t seed = 14695981039346656037ull;
static constexpr std::size_t prime = 1099511628211ull;

static auto hash_view(const std::string_view value) noexcept -> std::size_t {
    return append(seed, value);
}

static auto append(std::size_t hash, const std::string_view value) noexcept -> std::size_t {
    for (const auto ch : value) {
        hash ^= static_cast<unsigned char>(ch);
        hash *= prime;
    }
    return hash;
}
```

**FNV-1a 特点**:
- 逐字节混合，避免依赖输入长度
- 高位快速传播，分布均匀
- 无内存访问，适合小字符串

### is_transparent 标记

C++ 标准库通过 `is_transparent` 类型别名检测异构查找支持：

```cpp
unordered_map<string, Value, transparent_hash, transparent_equal> map;

// 异构查找：无需构造临时 string
map.find(string_view("key"));  // 零分配查找

// 传统方式（无 transparent）
unordered_map<string, Value> map2;
map2.find(string("key"));      // 需构造临时 string
```

## transparent_equal 结构

```cpp
struct transparent_equal {
    using is_transparent = void;  // 启用异构查找标记

    auto operator()(const std::string_view left, const std::string_view right) const noexcept -> bool;
    auto operator()(const memory::string &left, const std::string_view right) const noexcept -> bool;
    auto operator()(const std::string_view left, const memory::string &right) const noexcept -> bool;
    auto operator()(const memory::string &left, const memory::string &right) const noexcept -> bool;
};
```

提供四种 `operator()` 重载覆盖所有类型组合。

## 使用示例

### 在 router 中的使用

```cpp
template <typename Value>
using hash_map = memory::unordered_map<memory::string, Value, string_hash, string_equal>;

using reverse_map = hash_map<tcp::endpoint>;  // 反向路由表

// 查找时直接使用 string_view
auto route = reverse_map_.find(host);  // host 是 string_view，零分配
```

### 在 cache 中的使用

```cpp
using cache_map = memory::unordered_map<
    memory::string,
    std::pair<cache_entry, lru_list::iterator>,
    transparent_hash,
    transparent_equal>;

// 查找缓存键 "domain:qtype"
auto key_view = make_key_view(domain, qt, buffer);  // 在外部缓冲区构造视图
auto it = entries_.find(key_view);  // 零分配查找
```

### 在 coalescer 中的使用

```cpp
using flight_hash_map = memory::unordered_map<
    std::string_view,
    flight_iterator,
    transparent_hash,
    transparent_equal>;

flight_map_.emplace(std::string_view(flight_it->key), flight_it);
auto it = flight_map_.find(key_view);  // 零分配查找
```

## 性能优势

| 操作 | 传统方式 | 透明哈希 |
|------|----------|----------|
| `map.find("key")` | 构造临时 `string` | 直接使用 `string_view` |
| 堆分配 | 1 次 | 0 次 |
| 内存拷贝 | 字符串内容 | 无 |

**热路径优化**: DNS 解析和路由查找是高频操作，透明哈希避免了每次查找的临时对象构造开销。

## 参见

- [[core/resolve/router|string_hash]] — router 中使用的透明哈希
- [[core/resolve/dns/detail/cache|cache]] — DNS 缓存（使用 transparent_hash）
- [[core/resolve/dns/detail/coalescer|coalescer]] — 请求合并器（使用 transparent_hash）