---
title: "coalescer.hpp — 请求合并器"
source: "include/prism/resolve/dns/detail/coalescer.hpp"
module: "resolve"
type: api
tags: [resolve, dns, coalescer, 请求合并, 并发]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/dns/dns
  - resolve/dns/detail/transparent
  - memory/container
---

# coalescer.hpp

> 源码: `include/prism/resolve/dns/detail/coalescer.hpp`
> 模块: [[resolve|Resolve]] / dns / detail

## 概述

实现请求合并模式（Request Coalescing），用于将同一目标的并发请求合并为单次操作。当多个协程同时请求同一资源时，仅执行一次实际操作，其他协程等待结果复用。这在 DNS 解析等场景中特别有用，可以有效降低上游服务器压力。

核心机制：
- **请求标识**：通过键字符串（`"host:port"` 格式）唯一标识一个请求
- **请求跟踪**：使用 `flight` 结构体跟踪正在进行的请求
- **等待机制**：使用永不超时的 `steady_timer` 挂起等待的协程
- **结果广播**：请求完成时通过 `timer.cancel()` 通知所有等待者
- **延迟清理**：使用 `pending_cleanup` 标记避免迭代器失效

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[resolve/dns/detail/transparent|transparent]] | 透明哈希与比较器 |
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 被依赖 | [[resolve/dns/dns|dns]] | resolver_impl 使用合并器 |

## 命名空间

`psm::resolve::dns::detail`

---

## 类: coalescer

### 概述

请求合并器。内部维护一个 flight 列表和一个哈希索引，通过键快速查找正在进行的请求。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::resource_pointer` | `mr_` | 内存资源 |
| `flight_list` | `flights_` | 请求合并列表（`memory::list<flight>`） |
| `flight_hash_map` | `flight_map_` | 请求合并索引（键→迭代器映射） |

---

## 内部结构体: flight

### 概述

请求合并记录，用于跟踪正在进行的请求，支持多个协程等待同一请求完成。包含一个永不超时的定时器，用于挂起等待的协程直到请求完成。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `key` | 查找键（`"host:port"` 格式） |
| `net::steady_timer` | `timer` | 等待定时器（设为 `time_point::max()`） |
| `std::size_t` | `waiters` | 等待者计数 |
| `bool` | `ready` | 是否已完成 |
| `bool` | `pending_cleanup` | 是否待清理 |

---

### 函数: flight::flight()

- **功能说明**: 构造请求合并记录，初始化查找键和等待定时器。定时器设为永不超时（`time_point::max()`），用于挂起等待的协程直到请求完成时通过 `timer.cancel()` 唤醒。
- **签名**:
  ```cpp
  explicit flight(memory::string value, const net::any_io_executor &executor);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `value` | `memory::string` | 查找键字符串 |
  | `executor` | `const net::any_io_executor &` | 执行器，用于创建等待定时器 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `coalescer::find_or_create()` 在创建新 flight 时调用
- **知识域**: [[resolve/dns/detail/coalescer|请求合并]]

---

### 函数: coalescer::coalescer()

- **功能说明**: 构造请求合并器，使用指定的内存资源初始化内部的请求合并列表和索引。
- **签名**:
  ```cpp
  explicit coalescer(const memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | 内存资源，用于内部存储分配 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl]] 构造函数中初始化
- **知识域**: [[resolve/dns/detail/coalescer|请求合并]]

---

### 函数: coalescer::make_key()

- **功能说明**: 构造查找键字符串，将主机名和端口号拼接为 `"host:port"` 格式的键字符串，用于唯一标识一个请求目标。使用 PMR 分配器，预计算长度一次分配完成。
- **签名**:
  ```cpp
  [[nodiscard]] auto make_key(const std::string_view host, const std::string_view port) const -> memory::string;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `host` | `std::string_view` | 主机名 |
  | `port` | `std::string_view` | 服务端口 |

- **返回值**: `memory::string` — 格式为 `"host:port"` 的键字符串
- **调用（向下）**: 无（字符串拼接）
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 构造合并键时调用
- **知识域**: [[resolve/dns/detail/coalescer|请求合并]]

---

### 函数: coalescer::find_or_create()

- **功能说明**: 查找或创建请求合并记录。若该键对应的请求正在进行中，返回现有记录（`second=false`）；否则创建新的请求记录并插入索引（`second=true`）。键存储在 flight 对象中，确保哈希表中的 `string_view` 键指向有效的内存。
- **签名**:
  ```cpp
  auto find_or_create(const memory::string &key, const net::any_io_executor &executor)
      -> std::pair<flight_iterator, bool>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `key` | `const memory::string &` | 查找键 |
  | `executor` | `const net::any_io_executor &` | 执行器，用于创建等待定时器 |

- **返回值**: `std::pair<flight_iterator, bool>` — 迭代器和是否为新创建的标志
- **调用（向下）**: `flight_map_.find()` → `flights_.emplace_back()` → `flight_map_.emplace()`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在请求合并阶段调用
- **知识域**: [[resolve/dns/detail/coalescer|请求合并]]、[[resolve/dns/detail/transparent|透明哈希]]

---

### 函数: coalescer::cleanup_flight() [static]

- **功能说明**: 标记请求合并记录待清理。当请求已完成（`ready=true`）且无等待者（`waiters==0`）时，标记为 `pending_cleanup` 状态。实际删除操作在 `flush_cleanup()` 中执行，避免迭代器失效。
- **签名**:
  ```cpp
  static void cleanup_flight(const flight_iterator flight);
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `flight` | `const flight_iterator` | 请求合并记录的迭代器 |

- **返回值**: 无
- **调用（向下）**: 无（仅设置标志位）
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在请求完成和等待者唤醒后调用
- **知识域**: [[resolve/dns/detail/coalescer|请求合并]]

---

### 函数: coalescer::flush_cleanup()

- **功能说明**: 执行延迟清理，删除所有标记为 `pending_cleanup` 的 flight 记录。遍历 flights 列表，对每个待清理记录同时删除哈希索引和列表节点。应在安全时机调用（如下一次请求开始前）。
- **签名**:
  ```cpp
  void flush_cleanup();
  ```
- **参数**: 无
- **返回值**: 无
- **调用（向下）**: `flight_map_.erase()` + `flights_.erase()`
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在每次查询开始时调用
- **知识域**: [[resolve/dns/detail/coalescer|请求合并]]

---

## 调用链总览

```
[[resolve/dns/dns|resolver_impl::query_pipeline()]]
  ├── coalescer_.flush_cleanup()         → 清理已完成的 flight
  ├── coalescer_.make_key(host, port)    → 构造 "host:port" 键
  ├── coalescer_.find_or_create(key)     → 查找或创建 flight
  │   ├── 新请求 (is_new=true):  执行上游查询 → ready=true → timer.cancel()
  │   └── 已有请求 (is_new=false): ++waiters → timer.async_wait() → --waiters
  └── coalescer::cleanup_flight(it)      → 标记待清理
```

---

## 知识域

- [[resolve/dns/detail/coalescer|请求合并]]
- [[ref/programming/c++23-coroutines|并发控制]]
- [[resolve/dns/detail/transparent|透明哈希]]
