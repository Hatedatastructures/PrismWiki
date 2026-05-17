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
> 模块: [[resolve|resolve]] / [[resolve/dns|dns]] / detail

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