---
title: "cache — DNS 结果缓存"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/detail/cache.hpp"
module: "resolve/dns/detail"
type: component
tags: [dns, cache, ttl, serve-stale, negative-cache]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/dns
  - core/resolve/dns/detail/format
  - core/resolve/dns/detail/transparent
---

# cache — DNS 结果缓存

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/detail/cache.hpp`
> 模块: [[resolve|resolve]] / [[resolve/dns|dns]] / detail

## 组件定位

`cache` 提供 DNS 解析结果的内存缓存，支持正向缓存（IP 地址列表）和负缓存（解析失败标记）。缓存策略包括 TTL 过期、FIFO 淘汰和 serve-stale 模式。

## cache_entry 结构

```cpp
struct cache_entry {
    memory::vector<net::ip::address> ips;           // 解析结果 IP 地址列表
    uint32_t ttl{0};                                // 原始 TTL（秒）
    std::chrono::steady_clock::time_point expire;   // 过期时间
    std::chrono::steady_clock::time_point inserted; // 插入时间（用于 FIFO 淘汰）
    bool failed{false};                             // 负缓存标记
};
```

## cache 类

### 构造参数

```cpp
explicit cache(memory::resource_pointer mr = memory::current_resource(),
               std::chrono::seconds ttl = std::chrono::seconds(120),
               std::size_t max_entries = 10000,
               bool serve_stale = true);
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mr` | `current_resource()` | PMR 内存资源 |
| `ttl` | `120s` | 默认缓存 TTL |
| `max_entries` | `10000` | 最大条目数（触发 FIFO 淘汰） |
| `serve_stale` | `true` | 过期后仍返回旧数据 |

### 缓存键格式

```
"domain:qtype_number"
例如: "www.example.com:1" (A 记录)
      "www.example.com:28" (AAAA 记录)
```

通过 `transparent_hash` 实现零分配查找，避免构造临时 `memory::string`。

### 核心方法

#### get — 查找缓存

```cpp
[[nodiscard]] auto get(std::string_view domain, qtype qt)
    -> std::optional<memory::vector<net::ip::address>>;
```

返回值语义：
| 返回值 | 含义 |
|--------|------|
| `nullopt` | 未命中 |
| 空 `vector` | 负缓存命中（解析失败） |
| 非空 `vector` | 正向缓存命中 |

**查找逻辑**:
1. 构造缓存键 `"domain:qtype"`
2. 查找 `entries_`
3. 未找到 → `nullopt`
4. 未过期 → 返回 `ips`（若 `failed=true` 则返回空 vector）
5. 已过期 + `serve_stale=true` → 返回旧数据（调用方应触发后台刷新）
6. 已过期 + `serve_stale=false` → 删除条目，返回 `nullopt`

#### put — 写入正向缓存

```cpp
void put(std::string_view domain, qtype qt,
         const memory::vector<net::ip::address> &ips,
         uint32_t ttl_seconds);
```

流程：
1. 计算过期时间 `expire = now + ttl`
2. 创建 `cache_entry`
3. 插入 `entries_`
4. 若条目数超过 `max_entries_` → FIFO 淘汰最旧条目

#### put_negative — 写入负缓存

```cpp
void put_negative(std::string_view domain, qtype qt,
                   std::chrono::seconds negative_ttl = std::chrono::seconds(30));
```

记录解析失败的域名，在 `negative_ttl` 期间直接返回空结果。

**用途**: 防止对不可达域名的重复查询造成上游压力。

#### evict_expired — 清理过期

```cpp
void evict_expired();
```

遍历缓存表，删除所有已过期的条目。

## 缓存策略

### TTL 过期

每个条目存储：
- `expire` — 过期时间点
- `ttl` — 原始 TTL 值

过期条目处理：
- `serve_stale=true` — 返回旧数据 + 标记需刷新
- `serve_stale=false` — 立即删除

### FIFO 淘汰

当缓存条目数超过 `max_entries_`：
- 按 `inserted` 时间排序
- 删除最旧的条目
- 保持缓存大小在限制内

### 负缓存

解析失败的记录：
- `failed=true` 标记
- 较短 TTL（默认 30s）
- 返回空 vector 区分于未命中

**优势**: 避免对 NXDOMAIN 响应的重复查询，减轻上游负载。

## 内部结构

### LRU 链表

```cpp
using lru_list = memory::list<memory::string>;
lru_list lru_order_;  // 头部=最近访问，尾部=最旧
```

### 缓存表

```cpp
using cache_map = memory::unordered_map<
    memory::string,
    std::pair<cache_entry, lru_list::iterator>,
    transparent_hash,
    transparent_equal>;
cache_map entries_;
```

每个条目存储：
- `cache_entry` — 缓存数据
- `lru_list::iterator` — LRU 链表位置指针

## 私有辅助方法

### make_key

```cpp
auto make_key(std::string_view domain, qtype qt) const -> memory::string;
```

拼接 `"domain:qtype_number"`，用于持久存储。

### make_key_view

```cpp
static auto make_key_view(std::string_view domain, qtype qt, std::span<char> buffer)
    -> std::string_view;
```

在外部缓冲区中构造键视图，不产生堆分配。

**用途**: `get()` 查找时使用 `make_key_view` 实现零分配。

## 使用示例

```cpp
cache dns_cache(mr, 120s, 10000, true);

// 查找缓存
auto result = dns_cache.get("www.example.com", qtype::a);
if (result.has_value()) {
    if (result->empty()) {
        // 负缓存命中
    } else {
        // 正向缓存命中，使用 IPs
    }
} else {
    // 未命中，发起 DNS 查询
    auto ips = co_await upstream.resolve(domain, qtype::a);
    if (ips.empty()) {
        dns_cache.put_negative(domain, qtype::a);  // 写入负缓存
    } else {
        dns_cache.put(domain, qtype::a, ips, ttl); // 写入正向缓存
    }
}
```

## 调用链

```
resolver::resolve(host)
  │
  ├─→ cache::get(host, A)
  │     ├─→ 未命中 → 继续查询
  │     ├─→ 正向命中 → 返回 IPs
  │     └─→ 负缓存命中 → 返回空列表
  │
  ├─→ upstream::resolve()
  │
  └→ cache::put(host, A, ips, ttl)
        ├─→ 创建 entry
        ├─→ 插入 entries_
        └─→ FIFO 淘汰（若超限）
```

## 参见

- [[core/resolve/dns/dns|resolver]] — DNS 解析器接口
- [[core/resolve/dns/detail/format|qtype]] — 查询类型枚举
- [[core/resolve/dns/detail/transparent|transparent_hash]] — 透明哈希