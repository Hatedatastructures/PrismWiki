---
title: "coalescer -- 请求合并器"
layer: core
source: "include/prism/resolve/dns/detail/coalescer.hpp"
module: "resolve/dns/detail"
type: component
tags: [dns, coalescer, request-merge, concurrent]
created: 2026-05-17
updated: 2026-05-28
related:
  - core/resolve/dns/dns
  - core/resolve/dns/detail/transparent
---

# coalescer -- 请求合并器

> 源码位置: `include/prism/resolve/dns/detail/coalescer.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]] / detail

## 组件定位

`coalescer` 实现请求合并模式（Request Coalescing），将同一目标的并发请求合并为单次操作。当多个协程同时请求同一资源时，仅执行一次实际操作，其他协程等待结果后复用。

**典型场景**: DNS 解析中，多个会话同时请求解析同一域名，合并为单次上游查询。50 个并发请求解析同一域名时，合并后仅 1 次上游查询（减少 97% 以上）。

## 接口一览

### flight 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `key` | `memory::string` | 查找键，格式 `"host:port"` |
| `timer` | `net::steady_timer` | 永不超时的等待定时器 |
| `waiters` | `std::size_t` | 等待者计数 |
| `ready` | `bool` | 请求是否已完成 |
| `pending_cleanup` | `bool` | 延迟清理标记 |

### coalescer 方法

| 方法 | 签名 | 说明 |
|------|------|------|
| 构造 | `coalescer(resource_pointer mr)` | 初始化 flights 列表和索引 |
| `make_key` | `(host, port) -> string` | 返回 `"host:port"` 格式键 |
| `find_create` | `(key, executor) -> pair<iterator, bool>` | 查找或创建请求记录；`true` 表示新创建 |
| `cleanup_flight` | `(iterator) -> void` | 当 `ready && waiters==0` 时标记 `pending_cleanup` |
| `flush_cleanup` | `() -> void` | 删除所有 `pending_cleanup` 记录 |

### 数据结构

- `flights_` -- `memory::list<flight>`，请求合并列表
- `flight_map_` -- `memory::unordered_map<string_view, iterator, transparent_hash, transparent_equal>`，请求合并索引

## 工作流程

1. 调用方通过 `find_create(key, executor)` 获取请求记录
2. 若是新请求（返回 `true`）：发起实际 DNS 查询，完成后设 `ready=true`，调用 `timer.cancel()` 唤醒所有等待者，调用 `cleanup_flight`
3. 若已有请求（返回 `false`）：递增 `waiters`，`co_await timer.async_wait()` 挂起等待，被唤醒后递减 `waiters`，调用 `cleanup_flight`
4. 安全时机调用 `flush_cleanup()` 清除待清理记录

## 关键设计决策

### 为什么使用永不超时的 timer

**问题**: 等待协程需要一个挂起机制，但 DNS 查询已有自己的超时。

**选择**: `timer` 设为 `time_point::max()`，仅通过 `timer.cancel()` 唤醒等待者，不引入额外超时。

**后果**: 请求合并不干扰上游超时逻辑。但如果主请求未调用 `cancel()`，等待者将永久挂起。

**约束**: 等待的协程必须最终被通知，否则永久挂起。

### 为什么使用 pending_cleanup 延迟清理

**问题**: 在迭代 `flights_` 列表时直接删除条目会导致迭代器失效。

**选择**: 两阶段清理 -- `cleanup_flight` 仅设置 `pending_cleanup=true`，`flush_cleanup` 在安全时机集中删除。删除时同时清除 `flight_map_` 中对应的键。

**后果**: 等待者可以安全地在迭代中处理结果，不会有迭代器失效风险。

### 为什么 flight_map_ 使用 string_view 键

**问题**: 需要一个索引来快速查找进行中的请求。

**选择**: `flight_map_` 的 `string_view` 键指向 `flights_` 中 `flight::key` 的内存，避免重复存储键字符串。哈希表只持有视图，无额外分配。

**后果**: `flight` 的生命周期必须保证视图有效。仅在 `flush_cleanup` 中同时删除列表条目和索引条目。

### 键的合并粒度

去重键为 `"host:port"`。相同域名但不同端口的请求视为不同请求。

## 约束

| 类型 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| 线程安全 | 非线程安全，应在单个 `io_context` 线程中使用 | 数据竞争 | 单线程事件循环天然保证串行访问 |
| 资源泄漏 | 安全时机必须调用 `flush_cleanup()` | 已完成请求无法释放 | `coalescer.hpp:147-162` |
| 通知保证 | 主请求完成后必须调用 `timer.cancel()` | 等待者永久挂起 | `coalescer.hpp:60` |

## 调用链

```
resolver::resolve(host)
  |
  +-> coalescer::find_create("host:A", executor)
  |     +-> 新请求 -> upstream.resolve() -> ready=true -> timer.cancel() -> cleanup_flight()
  |     +-> 已有请求 -> waiters++ -> co_await timer.async_wait() -> waiters-- -> cleanup_flight()
  +-> flush_cleanup() 删除待清理记录
```

## 参见

- [[core/resolve/dns/dns|resolver]] -- DNS 解析器接口
- [[core/resolve/dns/detail/transparent|transparent_hash]] -- 透明哈希
