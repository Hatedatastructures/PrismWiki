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
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]] / detail

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

---

## 缓存机制详解

### 缓存架构

```
┌──────────────────────────────────────────────────────────────┐
│                        cache                                  │
│                                                              │
│  entries_ (hash_map):                                        │
│  ┌─────────────────────────────────────────────────┐        │
│  │ key (string) → (cache_entry, lru_iterator)      │        │
│  │                                                  │        │
│  │ key 格式: "domain:qtype"                         │        │
│  │   "www.example.com:1"   (A 记录)                │        │
│  │   "www.example.com:28"  (AAAA 记录)             │        │
│  └─────────────────────────────────────────────────┘        │
│                                                              │
│  lru_order_ (list):                                         │
│  ┌─────────────────────────────────────────────────┐        │
│  │ [head=最近访问] ... [tail=最旧]                  │        │
│  │                                                  │        │
│  │ 用于 FIFO 淘汰：超出 max_entries 时删除 tail      │        │
│  └─────────────────────────────────────────────────┘        │
│                                                              │
│  透明查找:                                                   │
│    get("www.example.com", qtype::a)                         │
│      ↓                                                      │
│    make_key_view → 外部缓冲区构造 string_view                │
│      ↓                                                      │
│    entries_.find(key_view) → 零堆分配查找                     │
└──────────────────────────────────────────────────────────────┘
```

### 缓存键设计

缓存键格式为 `"domain:qtype_number"`：

```cpp
static auto make_key(std::string_view domain, qtype qt) -> memory::string {
    memory::string key;
    key.reserve(domain.size() + 3);  // ":28" max
    key.append(domain);
    key.push_back(':');
    // 将 qtype 枚举值转为字符串
    auto qtype_num = static_cast<int>(qt);
    if (qtype_num >= 10) {
        key.push_back('0' + qtype_num / 10);
        key.push_back('0' + qtype_num % 10);
    } else {
        key.push_back('0' + qtype_num);
    }
    return key;
}
```

**常见 qtype 编号**:

| qtype | 编号 | 缓存键示例 |
|-------|------|-----------|
| A | 1 | `example.com:1` |
| AAAA | 28 | `example.com:28` |
| CNAME | 5 | `example.com:5` |
| MX | 15 | `example.com:15` |
| TXT | 16 | `example.com:16` |

## TTL 管理

### TTL 来源

1. **DNS 响应中的 TTL**: 权威 DNS 服务器在响应中指定的 TTL
2. **默认 TTL**: 当响应中无 TTL 时使用 `config::cache_ttl`
3. **TTL 钳制**: 最终 TTL 在 `[ttl_min, ttl_max]` 范围内

```
原始 TTL (来自 DNS 响应)
    ↓
钳制: clamped = max(ttl_min, min(ttl, ttl_max))
    ↓
存储: entry.ttl = clamped
    ↓
过期时间: entry.expire = now + clamped
```

### TTL 钳制逻辑

```cpp
auto clamp_ttl(uint32_t original_ttl) -> uint32_t {
    if (original_ttl < cfg.ttl_min)
        return cfg.ttl_min;  // 防止 TTL=0 导致无缓存
    if (original_ttl > cfg.ttl_max)
        return cfg.ttl_max;  // 防止数据过度陈旧
    return original_ttl;
}
```

**示例**:
```
原始 TTL = 5s    → 钳制后 = 60s (ttl_min)
原始 TTL = 86400 → 钳制后 = 86400 (在范围内)
原始 TTL = 604800 (7天) → 钳制后 = 86400 (ttl_max)
原始 TTL = 30s   → 钳制后 = 60s (ttl_min)
```

### TTL 衰减与刷新

```
缓存条目生命周期:

  inserted ───────────────────── expire ────────── serve_stale_end
     │                              │                    │
     │  有效数据                     │  过期但可服务       │
     │  (get 返回 + 更新 LRU)       │  (get 返回旧数据)   │  删除
     │                              │                    │
     └──────────────────────────────┼────────────────────┤
                                    │                    │
                              后台刷新触发点        最终过期
```

## serve-stale 机制

### 什么是 serve-stale

当缓存条目过期后，`serve_stale=true` 时仍可返回过期数据，同时触发后台刷新：

```cpp
auto get(domain, qt) -> optional<vector<ip::address>> {
    auto it = entries_.find(key);
    if (it == entries_.end()) return nullopt;  // 未命中

    if (it->second.expire > now) {
        // 未过期 → 正常返回 + 更新 LRU
        lru_order_.splice(lru_order_.begin(), lru_order_, it->second.lru_it);
        return it->second.ips;
    }

    // 已过期:
    if (serve_stale_) {
        // 返回旧数据 + 标记需刷新
        return it->second.ips;  // 调用方应触发后台刷新
    } else {
        // 删除过期条目
        entries_.erase(it);
        return nullopt;
    }
}
```

### serve-stale 时序

```
T0: 缓存写入，expire = T0 + 120s
T120: 缓存过期
T121: 用户请求 → get 返回旧数据 (serve-stale)
     同时触发后台刷新
T121+RTT: 后台刷新完成 → 缓存更新
     expire = T121+RTT + new_ttl
```

### serve-stale 的优势

| 场景 | 无 serve-stale | 有 serve-stale |
|------|---------------|----------------|
| DNS 上游临时不可用 | 请求阻塞/失败 | 返回旧数据，继续服务 |
| 网络抖动 | 解析中断 | 服务不中断 |
| 上游响应慢 | 等待超时 | 立即返回旧数据 |

### 负缓存 TTL

解析失败的记录使用独立的 `negative_ttl`:

```cpp
void put_negative(domain, qt, negative_ttl = 30s) {
    cache_entry entry;
    entry.failed = true;
    entry.ips = {};  // 空列表
    entry.expire = now + negative_ttl;
    entries_[key] = {entry, lru_order_.insert(lru_order_.begin(), key)};
}
```

**负缓存用途**:
- 防止对不可达域名的重复查询
- NXDOMAIN 响应也有 TTL（来自 DNS 响应中的 SOA 记录）
- 默认 30s 平衡了减少查询和数据新鲜度

## FIFO 淘汰策略

### 淘汰触发条件

```
put() → 插入新条目
    ↓
if entries_.size() > max_entries:
    触发淘汰
```

### 淘汰过程

```
┌───────────────────────────────────────────┐
│ LRU 链表:                                  │
│ [最近访问] → A → B → C → D → [最旧: E]    │
│                                           │
│ entries_.size() > max_entries → 淘汰 E    │
│   │                                       │
│   ├── 从 lru_order_ 移除 tail             │
│   ├── 从 entries_ 移除对应 key            │
│   └── 释放内存                             │
│                                           │
│ 结果: [最近访问] → A → B → C → D          │
└───────────────────────────────────────────┘
```

### 为什么选择 FIFO 而非 LRU

当前实现使用 LRU 链表但执行 FIFO 淘汰（淘汰最旧的而非最少使用的）：

| 策略 | 优点 | 缺点 |
|------|------|------|
| FIFO | 简单，O(1) 淘汰 | 不区分访问频率 |
| LRU | 淘汰最少使用的 | 需要每次 get 时更新链表 |
| LFU | 考虑访问频率 | 实现复杂，空间开销大 |

**当前设计**: 插入时放到链表头部（最近），淘汰时从尾部（最旧）移除。这是一个"插入顺序 FIFO"策略，实现简单且对 DNS 缓存场景足够有效。

### 内存占用估算

```
每条缓存条目:
  key:      域名长度 + 3 字节
  entry:    vector<ip> + 时间戳 + 标志 ≈ 64 字节
  lru_it:   迭代指针 8 字节

单条条目 ≈ 100-200 字节 (取决于域名长度和 IP 数)

10000 条目 ≈ 1-2 MB
100000 条目 ≈ 10-20 MB
```

## 缓存操作总结

| 操作 | 方法 | 触发条件 | 时间复杂度 |
|------|------|----------|-----------|
| 查找 | `get()` | 每次 resolve 调用 | O(1) 平均 |
| 插入 | `put()` | 上游查询成功 | O(1) 平均 |
| 负插入 | `put_negative()` | 上游查询失败 | O(1) 平均 |
| 淘汰 | put() 内部 | size > max_entries | O(1) |
| 清理 | `evict_expired()` | 定期/手动 | O(N) |