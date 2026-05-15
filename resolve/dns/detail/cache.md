---
title: "cache.hpp — DNS 结果缓存"
source: "include/prism/resolve/dns/detail/cache.hpp"
module: "resolve"
type: api
tags: [resolve, dns, cache, 缓存, LRU, 负缓存]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/dns/dns
  - resolve/dns/detail/format
  - resolve/dns/detail/transparent
  - memory/container
---

# cache.hpp

> 源码: `include/prism/resolve/dns/detail/cache.hpp` + `src/prism/resolve/dns/detail/cache.cpp`
> 模块: [[resolve|Resolve]] / dns / detail

## 概述

提供 DNS 解析结果的内存缓存，支持正向缓存（IP 地址列表）和负缓存（解析失败标记）。缓存键采用 `"domain:qtype_number"` 格式，通过 [[resolve/dns/detail/transparent|transparent_hash]] / [[resolve/dns/detail/transparent|transparent_equal]] 实现异构查找，避免构造临时键对象的额外开销。

缓存策略包括：
- **TTL 过期**：条目在过期后可配置为 serve-stale 模式返回旧数据，同时允许调用方触发后台刷新
- **LRU 淘汰**：当缓存条目数超过上限时按 LRU 策略淘汰最旧的条目
- **负缓存**：解析失败的记录会以较短的 TTL 缓存，防止对不可达域名的重复解析请求造成上游压力

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[resolve/dns/detail/format|format]] | qtype 枚举 |
| 依赖 | [[resolve/dns/detail/transparent|transparent]] | 透明哈希与比较器 |
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 被依赖 | [[resolve/dns/dns|dns]] | resolver_impl 使用缓存 |

## 命名空间

`psm::resolve::dns::detail`

---

## 结构体: cache_entry

### 概述

DNS 缓存条目，存储单次 DNS 解析的结果及其元数据，包括解析得到的 IP 地址列表、原始 TTL、过期时间、插入时间和负缓存标记。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::vector<net::ip::address>` | `ips` | 解析结果 IP 地址列表 |
| `uint32_t` | `ttl` | 原始 TTL（秒） |
| `std::chrono::steady_clock::time_point` | `expire` | 过期时间 |
| `std::chrono::steady_clock::time_point` | `inserted` | 插入时间（用于 LRU 淘汰） |
| `bool` | `failed` | 负缓存标记 |

---

### 函数: cache_entry::cache_entry()

- **功能说明**: 构造 DNS 缓存条目，使用指定内存资源初始化 IP 地址列表容器。
- **签名**:
  ```cpp
  explicit cache_entry(memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | 内存资源指针 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `cache::put()` 和 `cache::put_negative()` 内部构造条目
- **知识域**: [[memory/container|PMR 容器]]

---

## 类: cache

### 概述

DNS 结果缓存容器，负责存储和检索 DNS 解析结果。缓存键格式为 `"domain:qtype_number"`（如 `"www.example.com:1"`），通过 [[resolve/dns/detail/transparent|transparent_hash]] 的 `string_view` 重载实现零分配查找。

使用方式：
1. 调用 `get()` 查询缓存判断是否命中
2. 缓存未命中时发起 DNS 查询
3. 查询成功调用 `put()` 写入正向缓存
4. 查询失败调用 `put_negative()` 写入负缓存

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::resource_pointer` | `mr_` | 内存资源 |
| `std::chrono::seconds` | `default_ttl_` | 默认 TTL |
| `std::size_t` | `max_entries_` | 最大条目数 |
| `bool` | `serve_stale_` | serve-stale 模式开关 |
| `lru_list` | `lru_order_` | LRU 访问顺序链表（头部=最近访问） |
| `cache_map` | `entries_` | 缓存表（键→条目+LRU迭代器） |

---

### 函数: cache::cache()

- **功能说明**: 构造 DNS 缓存容器，初始化内存资源、默认 TTL、最大条目数和 serve-stale 模式。
- **签名**:
  ```cpp
  explicit cache(memory::resource_pointer mr = memory::current_resource(),
                 std::chrono::seconds ttl = std::chrono::seconds(120),
                 std::size_t max_entries = 10000, bool serve_stale = true);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | 内存资源指针 |
  | `ttl` | `std::chrono::seconds` | 默认缓存 TTL（秒） |
  | `max_entries` | `std::size_t` | 缓存最大条目数 |
  | `serve_stale` | `bool` | 是否在过期后返回旧数据 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl]] 构造函数中初始化
- **知识域**: [[resolve/dns/detail/cache|DNS 缓存]]

---

### 函数: cache::get()

- **功能说明**: 查找缓存。使用栈缓冲区构造查找键（零分配），通过 [[resolve/dns/detail/transparent|transparent_hash]] 异构查找。查找逻辑：未命中返回 `nullopt`；未过期返回 ips（负缓存返回空 vector）；已过期且 serve_stale 返回旧数据（调用方应触发后台刷新）；已过期且非 serve_stale 删除条目返回 `nullopt`。命中时自动更新 LRU 顺序（移到链表头部）。
- **签名**:
  ```cpp
  [[nodiscard]] auto get(std::string_view domain, qtype qt)
      -> std::optional<memory::vector<net::ip::address>>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 域名 |
  | `qt` | `qtype` | 查询类型 |

- **返回值**: `std::optional<memory::vector<net::ip::address>>` — `nullopt` 表示未命中；空 vector 表示负缓存命中；非空 vector 表示正向缓存命中
- **调用（向下）**: `make_key_view()` → `entries_.find()` → LRU 更新（`lru_order_.splice()`）
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在缓存查找阶段调用
- **知识域**: [[resolve/dns/detail/cache|DNS 缓存]]、[[resolve/dns/detail/transparent|透明哈希]]

---

### 函数: cache::put()

- **功能说明**: 写入正向缓存。使用栈缓冲区构造键（零分配），检查是否已存在（更新情况）或新插入。新插入时添加到 LRU 链表头部，超过 `max_entries_` 时按 LRU 策略淘汰链表尾部最旧条目。
- **签名**:
  ```cpp
  void put(std::string_view domain, qtype qt, const memory::vector<net::ip::address> &ips,
           uint32_t ttl_seconds);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 域名 |
  | `qt` | `qtype` | 查询类型 |
  | `ips` | `const memory::vector<net::ip::address> &` | 解析得到的 IP 地址列表 |
  | `ttl_seconds` | `uint32_t` | 缓存 TTL（秒） |

- **返回值**: 无
- **调用（向下）**: `make_key_view()` → `entries_.find()` → 更新或 `entries_.emplace()` → LRU 淘汰循环
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在 TTL 钳制后调用
- **知识域**: [[resolve/dns/detail/cache|DNS 缓存]]

---

### 函数: cache::put_negative()

- **功能说明**: 写入负缓存。记录解析失败的域名，在 `negative_ttl` 期间直接返回空结果，避免对不可达域名的重复查询造成上游压力。逻辑与 `put()` 类似，但设置 `failed=true`。
- **签名**:
  ```cpp
  void put_negative(std::string_view domain, qtype qt,
                    std::chrono::seconds negative_ttl = std::chrono::seconds(30));
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 域名 |
  | `qt` | `qtype` | 查询类型 |
  | `negative_ttl` | `std::chrono::seconds` | 负缓存 TTL（秒） |

- **返回值**: 无
- **调用（向下）**: `make_key_view()` → `entries_.find()` → 更新或 `entries_.emplace()`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在查询失败或 IP 全部被过滤时调用
- **知识域**: [[resolve/dns/detail/cache|DNS 缓存]]

---

### 函数: cache::evict_expired()

- **功能说明**: 清理过期条目。遍历缓存表，删除所有已过期的条目（包括 serve_stale 模式下的过期条目），同步删除 LRU 链表节点。由定时协程每 30 秒调用一次。
- **签名**:
  ```cpp
  void evict_expired();
  ```
- **参数**: 无
- **返回值**: 无
- **调用（向下）**: 遍历 `entries_` → `lru_order_.erase()` + `entries_.erase()`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl]] 中的定时清理协程每 30 秒调用
- **知识域**: [[resolve/dns/detail/cache|DNS 缓存]]

---

### 函数: cache::make_key() [private]

- **功能说明**: 构造缓存键（PMR string）。将域名和查询类型数值拼接为 `"domain:qtype_number"` 格式，如 `"www.example.com:1"`（A 记录）或 `"www.example.com:28"`（AAAA 记录）。
- **签名**:
  ```cpp
  [[nodiscard]] auto make_key(std::string_view domain, qtype qt) const -> memory::string;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 域名 |
  | `qt` | `qtype` | 查询类型 |

- **返回值**: `memory::string` — 格式为 `"domain:qtype_number"` 的 PMR 字符串
- **调用（向下）**: 无（字符串拼接）
- **被调用（向上）**: 内部使用（`make_key_view` 为更优替代）
- **知识域**: [[resolve/dns/detail/cache|DNS 缓存]]

---

### 函数: cache::make_key_view() [private, static]

- **功能说明**: 构造缓存键视图（零分配）。将域名和查询类型数值写入外部栈缓冲区并返回对应视图，不产生任何堆分配。使用 `std::snprintf` 写入数值部分。
- **签名**:
  ```cpp
  [[nodiscard]] static auto make_key_view(std::string_view domain, qtype qt, std::span<char> buffer)
      -> std::string_view;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 域名 |
  | `qt` | `qtype` | 查询类型 |
  | `buffer` | `std::span<char>` | 输出缓冲区（至少 260 字节） |

- **返回值**: `std::string_view` — 缓冲区中构造的键视图
- **调用（向下）**: 无（直接写入缓冲区）
- **被调用（向上）**: `get()`、`put()`、`put_negative()` 内部使用
- **知识域**: [[ref/memory/pmr|零分配优化]]

---

## 调用链总览

```
[[resolve/dns/dns|resolver_impl::query_pipeline()]]
  ├── cache::get()    → make_key_view() → entries_.find() → LRU 更新
  ├── cache::put()    → make_key_view() → entries_.emplace() → LRU 淘汰
  └── cache::put_negative() → make_key_view() → entries_.emplace()

定时清理协程 → cache::evict_expired() → 遍历删除过期条目
```

---

## 知识域

- [[resolve/dns/detail/cache|DNS 缓存]]
- [[resolve/dns/detail/cache|LRU 淘汰策略]]
- [[ref/memory/pmr|零分配优化]]
