---
title: "coalescer — 请求合并器"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/detail/coalescer.hpp"
module: "resolve/dns/detail"
type: component
tags: [dns, coalescer, request-merge, concurrent]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/dns
  - core/resolve/dns/detail/transparent
---

# coalescer — 请求合并器

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/detail/coalescer.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]] / detail

## 组件定位

`coalescer` 实现请求合并模式（Request Coalescing），将同一目标的并发请求合并为单次操作。当多个协程同时请求同一资源时，仅执行一次实际操作，其他协程等待结果后复用。

**典型场景**: DNS 解析中，多个会话同时请求解析同一域名，合并为单次上游查询。

## flight 结构

```cpp
struct flight {
    memory::string key;          // 查找键
    net::steady_timer timer;     // 等待定时器
    std::size_t waiters{0};      // 等待者计数
    bool ready{false};           // 是否已完成
    bool pending_cleanup{false}; // 是否待清理
};
```

**核心机制**:
- `timer` — 设为永不超时，用于挂起等待协程
- `waiters` — 记录等待中的协程数量
- `ready` — 标记请求是否已完成
- `pending_cleanup` — 延迟清理标记，避免迭代器失效

## coalescer 类

### 构造函数

```cpp
explicit coalescer(const memory::resource_pointer mr = memory::current_resource())
    : mr_(mr), flights_(mr), flight_map_(mr) {}
```

### 核心方法

#### make_key — 构造查找键

```cpp
auto make_key(const std::string_view host, const std::string_view port) const
    -> memory::string;
```

返回 `"host:port"` 格式的键字符串。

#### find_or_create — 查找或创建请求记录

```cpp
auto find_or_create(const memory::string &key, const net::any_io_executor &executor)
    -> std::pair<flight_iterator, bool>;
```

返回值：
- `{iterator, true}` — 新创建的请求
- `{iterator, false}` — 已存在的请求

**实现**:
```cpp
auto find_or_create(const memory::string &key, const net::any_io_executor &executor)
    -> std::pair<flight_iterator, bool>
{
    const std::string_view key_view(key);
    if (const auto it = flight_map_.find(key_view); it != flight_map_.end()) {
        return {it->second, false};  // 已存在，返回现有记录
    }

    flights_.emplace_back(key, executor);
    const auto flight_it = std::prev(flights_.end());
    flight_map_.emplace(std::string_view(flight_it->key), flight_it);
    return {flight_it, true};  // 新创建
}
```

#### cleanup_flight — 标记待清理

```cpp
static void cleanup_flight(const flight_iterator flight);
```

当请求完成且无等待者时，标记 `pending_cleanup=true`。实际删除在 `flush_cleanup` 中执行。

#### flush_cleanup — 执行延迟清理

```cpp
void flush_cleanup();
```

遍历 `flights_`，删除所有 `pending_cleanup=true` 的记录。

**延迟清理原因**: 避免在迭代过程中删除导致迭代器失效。

## 数据结构

### flights 列表

```cpp
using flight_list = memory::list<flight>;
flight_list flights_;  // 请求合并列表
```

### flight_map 索引

```cpp
using flight_hash_map = memory::unordered_map<
    std::string_view,
    flight_iterator,
    transparent_hash,
    transparent_equal>;
flight_hash_map flight_map_;  // 请求合并索引
```

**设计**: `flight_map_` 的键指向 `flights_` 中 `flight::key` 的 `string_view`，避免重复存储键字符串。

## 工作流程

### 发起请求

```cpp
auto [flight_it, is_new] = coalescer.find_or_create(key, executor);

if (is_new) {
    // 新请求：发起实际 DNS 查询
    flight_it->waiters = 1;
    auto result = co_await upstream.resolve(domain, qtype);
    flight_it->ready = true;
    flight_it->timer.cancel();  // 通知所有等待者
    coalescer.cleanup_flight(flight_it);
    co_return result;
} else {
    // 已有请求：等待结果
    flight_it->waiters++;
    co_await flight_it->timer.async_wait();
    flight_it->waiters--;
    // 复用结果（结果存储在外部上下文中）
    coalescer.cleanup_flight(flight_it);
    co_return result;
}
```

### 等待机制

`flight::timer` 设置为永不超时：

```cpp
timer.expires_at(std::chrono::steady_clock::time_point::max());
```

等待协程 `co_await timer.async_wait()` 会挂起，直到：
- 主请求完成时调用 `timer.cancel()`
- 取消唤醒所有等待者

## 关键设计决策

### timer 永不超时

不使用超时等待，因为：
- DNS 查询已有自己的超时机制
- 请求合并不应引入额外超时
- 主请求超时后通知等待者

### pending_cleanup 延迟清理

不立即删除已完成请求的原因：
- 等待者可能仍在处理结果
- 删除会导致 `flight_map_` 迭代器失效
- 标记后在下一次 `flush_cleanup` 中安全删除

### 键存储在 flight 中

`flight_map_` 使用 `string_view` 键，指向 `flight::key`：
- 键字符串存储在 `flight` 对象中
- 哈希表只持有视图，无额外分配
- `flight` 生命周期保证视图有效

## 使用注意事项

1. **单线程上下文**: 该组件不是线程安全的，应在单个线程中使用
2. **等待者必须被通知**: 主请求完成后必须调用 `timer.cancel()`
3. **及时清理**: 在安全时机调用 `flush_cleanup()` 防止内存泄漏

## 调用链

```
resolver::resolve(host)
  │
  ├─→ coalescer::find_or_create("host:A", executor)
  │     ├─→ 新请求 → 发起 upstream.resolve()
  │     │           → flight::ready = true
  │     │           → timer.cancel() 通知等待者
  │     │           → cleanup_flight()
  │     │
  │     └─→ 已有请求 → waiters++
  │                   → co_await timer.async_wait()
  │                   → waiters--
  │                   → cleanup_flight()
  │
  └→ flush_cleanup() 删除待清理记录
```

## 参见

- [[core/resolve/dns/dns|resolver]] — DNS 解析器接口
- [[core/resolve/dns/detail/transparent|transparent_hash]] — 透明哈希

---

## 查询合并机制

### 合并动机

在并发服务中，多个请求可能同时到达：

```
无合并:
  Request 1 ──▶ resolve("www.example.com") ──▶ upstream query
  Request 2 ──▶ resolve("www.example.com") ──▶ upstream query
  Request 3 ──▶ resolve("www.example.com") ──▶ upstream query

  → 3 次独立上游查询，3 倍延迟和带宽

有合并:
  Request 1 ──▶ coalescer ──▶ upstream query ──▶ 分发结果
  Request 2 ──▶ coalescer ──▶ wait ────────────────▶ 收到结果
  Request 3 ──▶ coalescer ──▶ wait ────────────────▶ 收到结果

  → 1 次上游查询，所有请求共享结果
```

### 合并数据结构

```cpp
struct flight {
    memory::string key;           // 合并键: "host:qtype"
    net::steady_timer timer;      // 等待通知定时器
    std::size_t waiters{0};       // 等待者数量
    bool ready{false};            // 是否已完成
    bool pending_cleanup{false};  // 是否待清理
};
```

**双索引结构**:
```
flights_ (list<flight>):
  ┌─────────────────────────────────────────┐
  │ [flight_1] → [flight_2] → [flight_3]   │
  │  key="a.com:1"   key="b.com:1"  ...     │
  └─────────────────────────────────────────┘

flight_map_ (hash_map<string_view, iterator>):
  ┌─────────────────────────────────────────┐
  │ "a.com:1" → iterator→flight_1           │
  │ "b.com:1" → iterator→flight_2           │
  └─────────────────────────────────────────┘
```

**设计原因**: `flight_map_` 使用 `string_view` 键指向 `flights_` 中的 `flight::key`，避免重复存储字符串。

## 去重策略

### 查找与创建

```cpp
auto find_or_create(const memory::string& key, const executor& exec)
    -> pair<flight_iterator, bool>
{
    // 1. 在 flight_map_ 中查找
    auto it = flight_map_.find(key);
    if (it != flight_map_.end()) {
        // 已存在 → 等待
        it->second->waiters++;
        return {it->second, false};
    }

    // 2. 不存在 → 创建新 flight
    flights_.emplace_back(key, exec);
    auto flight_it = std::prev(flights_.end());
    flight_map_.emplace(flight_it->key, flight_it);
    return {flight_it, true};  // 新请求，需要发起查询
}
```

### 等待者通知

```cpp
// 主查询完成:
flight->ready = true;
flight->timer.cancel();  // 取消定时器 → 唤醒所有等待者

// 等待者被唤醒:
co_await flight->timer.async_wait();
// timer.cancel() 导致 async_wait 立即完成 (operation_aborted)
```

**关键设计**: `timer` 设为永不超时（`time_point::max()`），确保只有 `cancel()` 才能唤醒等待者。

### 去重键

```cpp
auto make_key(host, port) -> memory::string {
    return host + ":" + port;
}
```

**合并粒度**: 按 `"host:port"` 去重。相同域名但不同端口的请求视为不同请求。

### 竞争条件处理

```
线程 A: find_or_create("a.com:1") → 未找到
线程 B: find_or_create("a.com:1") → 未找到 (竞争!)
线程 A: 创建 flight → 插入 map
线程 B: 创建 flight → 发现已有 → 返回已存在

→ 仅发起一次实际查询
```

**注意**: 该组件非线程安全，应在单个 `io_context` 线程中使用。Co-Asio 的单线程事件循环模型天然保证串行访问。

## 延迟清理机制

### 问题

如果在迭代 `flights_` 列表时删除条目，会导致迭代器失效：

```cpp
// 错误示例
for (auto it = flights_.begin(); it != flights_.end(); ) {
    if (it->ready && it->waiters == 0) {
        flights_.erase(it);  // 迭代器失效!
    }
}
```

### 解决方案: 两阶段清理

```
阶段 1: cleanup_flight(flight)
  └── 设置 flight.pending_cleanup = true
      (不执行删除，迭代器仍有效)

阶段 2: flush_cleanup()
  └── 遍历 flights_，删除所有 pending_cleanup 标记的条目
      同时从 flight_map_ 中移除对应键
```

```cpp
static void cleanup_flight(const flight_iterator flight) {
    if (flight->waiters == 0) {
        flight->pending_cleanup = true;
    }
}

void flush_cleanup() {
    // 先清除 flight_map_ 中的引用
    for (auto it = flights_.begin(); it != flights_.end(); ) {
        if (it->pending_cleanup) {
            flight_map_.erase(it->key);
            it = flights_.erase(it);
        } else {
            ++it;
        }
    }
}
```

**调用时机**: 在 `resolver::resolve()` 完成查询后调用 `flush_cleanup()`。

## 合并效果分析

### 并发场景

```
N 个并发请求解析同一域名:

无合并:  N 次上游查询, 总延迟 = N × RTT
有合并:  1 次上游查询, 总延迟 = 1 × RTT

合并效率 = 1/N (查询次数/请求数)
```

### 实际场景

```
Web 服务器启动时:
  50 个请求 → resolve("api.example.com")
  30 个请求 → resolve("cdn.example.com")
  20 个请求 → resolve("auth.example.com")

无合并: 100 次上游查询
有合并: 3 次上游查询 (减少 97%)
```

## 状态转换

```
flight 生命周期:

  ┌──────────┐    find_or_create     ┌──────────┐
  │  不存在   │ ────────────────────▶ │  创建中   │
  │ (新请求)  │                       │ (active) │
  └──────────┘                       └────┬─────┘
                                          │
                                   upstream.query()
                                          │
                                    ┌─────▼─────┐
                                    │  等待中    │
                                    │ (waiters) │
                                    └─────┬─────┘
                                          │
                                  timer.cancel()
                                   (结果就绪)
                                    ┌─────▼─────┐
                                    │  已完成    │
                                    │ (ready)   │
                                    └─────┬─────┘
                                          │
                              waiters=0 → cleanup_flight
                                    ┌─────▼─────┐
                                    │  待清理    │
                                    │ (pending) │
                                    └─────┬─────┘
                                          │
                                 flush_cleanup
                                    ┌─────▼─────┐
                                    │  已删除    │
                                    └───────────┘
```