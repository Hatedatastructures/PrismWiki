---
title: "Memory 模块"
module: "memory"
type: module
tags: [memory, pmr, 内存池, 容器, arena]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/connection/pool
  - agent/config
  - agent/worker/worker
  - resolve/router
  - fault/code
---

# Memory 模块

> 模块: [[memory|memory]]

## 概述

Memory 模块提供 PMR（Polymorphic Memory Resource）内存池和自定义容器。三级池化架构：thread_local_pool、session_pool、global_pool，减少锁竞争。热路径容器通过 PMR 分配器自动使用内存池，实现零堆分配目标。

---

## container.hpp — PMR 容器

> 源码: `include/prism/memory/container.hpp`
> 模块: [[memory|memory]]

### 概述

PMR 容器类型别名。定义使用 `std::pmr::polymorphic_allocator` 的容器类型，为项目提供统一的内存管理基础设施。所有容器类型均支持运行时切换内存资源，实现与自定义内存池的无缝集成。

### 命名空间

`psm::memory`

### 类型别名

| 类型 | 底层类型 | 说明 |
|------|----------|------|
| `resource` | `std::pmr::memory_resource` | 内存资源基类，配合 monotonic_buffer 使用具有良好的缓存友好性 |
| `resource_pointer` | `std::pmr::memory_resource*` | 内存资源指针类型，用于在函数间传递内存资源引用 |
| `allocator<T>` | `std::pmr::polymorphic_allocator<T>` | 多态内存分配器模板 |
| `synchronized_pool` | `std::pmr::synchronized_pool_resource` | 线程安全的池资源，内部使用互斥锁保护 |
| `unsynchronized_pool` | `std::pmr::unsynchronized_pool_resource` | 非线程安全的池资源，仅限单线程使用 |
| `monotonic_buffer` | `std::pmr::monotonic_buffer_resource` | 单调增长缓冲区资源，仅分配不释放 |
| `string` | `std::pmr::string` | PMR 字符串类型 |
| `vector<Value>` | `std::pmr::vector<Value>` | PMR 动态数组模板 |
| `list<Value>` | `std::pmr::list<Value>` | PMR 双向链表模板 |
| `map<Key,Value,Compare>` | `std::pmr::map<Key,Value,Compare>` | PMR 红黑树映射模板 |
| `unordered_map<K,V,Hash,Eq>` | `std::pmr::unordered_map<K,V,Hash,Eq>` | PMR 哈希映射模板 |
| `unordered_set<K,Hash,Eq>` | `std::pmr::unordered_set<K,Hash,Eq>` | PMR 哈希集合模板 |

### 函数: current_resource()

- **功能说明**: 获取当前默认内存资源。对 `std::pmr::get_default_resource()` 的包装，如果调用了 `system::enable_global_pooling()` 则返回 global_pool()，否则返回系统堆分配器。
- **签名**: `inline auto current_resource() -> resource_pointer`
- **参数**: 无
- **返回值**: `resource_pointer` — 当前默认内存资源指针，永不返回 nullptr
- **调用（向下）**: `std::pmr::get_default_resource()`
- **被调用（向上）**: `resolve::router` 构造、`protocol::analysis::target` 构造、`recognition::recognize()`、`multiplex::core` 构造、`pipeline::primitives`、`dns::upstream` 构造、`dns::resolver_impl` 构造、`dns::cache` 构造、`dns::format`、`shadowsocks::datagram`、`vless::relay`、`smux::craft`
- **知识域**: PMR 内存资源全局默认值

---

## pool.hpp — 内存池系统

> 源码: `include/prism/memory/pool.hpp`
> 模块: [[memory|memory]]

### 概述

PMR 内存池系统实现。提供全局和线程局部的内存池管理，以及基于内存池的对象分配基类和帧分配器。遵循热路径无分配、线程封闭和大小分类的设计原则。

### 结构体: policy

- **功能说明**: 内存策略配置。定义内存池的调优参数，针对代理服务器典型负载进行优化，平衡内存利用率、分配速度和内存峰值。
- **签名**: `struct policy`
- **参数**: 无（编译期常量）

| 字段 | 类型 | 值 | 说明 |
|------|------|-----|------|
| `max_blocks` | `std::size_t` | 256 | 每个 Chunk 包含的最大块数，降低此值可减少内存峰值 |
| `max_pool_size` | `std::size_t` | 16384 | 最大池化阈值（16KB），足以覆盖 HTTP Header 等典型对象 |

- **返回值**: 不适用
- **调用（向下）**: 无
- **被调用（向上）**: `system::global_pool()`、`system::thread_local_pool()`、`pooled_object::operator new/delete`
- **知识域**: 内存池调优参数

### 类: system

- **功能说明**: 全局内存系统管理器。提供全局单例的内存池访问接口，管理全局同步池、线程局部池和热路径池。所有方法均为静态方法，无需实例化。
- **签名**: `class system`

#### system::global_pool()

- **功能说明**: 获取全局线程安全池。适用于跨线程传递的对象和生命周期不确定的长期对象。使用 new 分配，确保在静态析构阶段后仍可用。
- **签名**: `static synchronized_pool *global_pool()`
- **参数**: 无
- **返回值**: `synchronized_pool*` — 全局同步池指针，永不返回 nullptr
- **调用（向下）**: `policy::max_pool_size`、`policy::max_blocks`、`std::pmr::synchronized_pool_resource` 构造
- **被调用（向上）**: `system::enable_global_pooling()`、`pooled_object<pool_type::global>::get_target_pool()`
- **知识域**: 全局内存池管理

#### system::thread_local_pool()

- **功能说明**: 获取线程局部内存池。返回线程局部的无锁内存池，适用于临时计算和单线程处理逻辑。使用 `thread_local` 存储，每个线程独立实例。
- **签名**: `static unsynchronized_pool *thread_local_pool()`
- **参数**: 无
- **返回值**: `unsynchronized_pool*` — 线程局部池指针，永不返回 nullptr
- **调用（向下）**: `policy::max_pool_size`、`policy::max_blocks`、`std::pmr::unsynchronized_pool_resource` 构造
- **被调用（向上）**: `system::hot_path_pool()`、`frame_arena` 构造、`pooled_object<pool_type::local>::get_target_pool()`、`worker::worker` 构造、`loader::build_account_directory()`、`pipeline::primitives`
- **知识域**: 线程局部内存池管理

#### system::hot_path_pool()

- **功能说明**: 获取热路径内存池。是 `thread_local_pool()` 的语义化别名，用于明确标识热路径场景。分配的对象生命周期必须与当前线程绑定，禁止跨线程传递。
- **签名**: `static unsynchronized_pool *hot_path_pool()`
- **参数**: 无
- **返回值**: `unsynchronized_pool*` — 线程局部池指针（与 thread_local_pool 相同）
- **调用（向下）**: `thread_local_pool()`
- **被调用（向上）**: 热路径对象分配
- **知识域**: 热路径内存分配语义

#### system::enable_global_pooling()

- **功能说明**: 启用全局池化策略。将默认内存资源设置为全局内存池，调用后所有未指定显式内存资源的 PMR 容器将自动使用 `global_pool()`。应在程序启动早期调用。
- **签名**: `static void enable_global_pooling()`
- **参数**: 无
- **返回值**: 无
- **调用（向下）**: `std::pmr::set_default_resource()`、`global_pool()`
- **被调用（向上）**: `main.cpp` 启动流程第一步
- **知识域**: PMR 全局池化初始化

### 枚举: pool_type

- **功能说明**: 内存池类型选择。用于 `pooled_object` 基类选择不同的内存池策略。
- **签名**: `enum class pool_type`
- **参数**: 无

| 值 | 说明 |
|----|------|
| `global` | 全局线程安全池，适用于跨线程传递对象 |
| `local` | 线程局部无锁池，适用于单线程热路径对象 |

- **返回值**: 不适用
- **调用（向下）**: 无
- **被调用（向上）**: `pooled_object<T, Type>` 模板参数
- **知识域**: 池类型策略

### 类模板: pooled_object

- **功能说明**: 对象池基类模板。通过重载 `operator new/delete` 使继承类自动使用内存池分配。小对象（<=16KB）使用指定池，大对象直通系统堆。使用 CRTP 惯用法。
- **签名**: `template <typename T, pool_type Type = pool_type::local> class pooled_object`

#### pooled_object::get_target_pool()

- **功能说明**: 获取目标内存池。根据池类型返回对应的内存池指针。
- **签名**: `static resource_pointer get_target_pool()`
- **参数**: 无
- **返回值**: `resource_pointer` — 全局池或线程局部池指针
- **调用（向下）**: `system::global_pool()` 或 `system::thread_local_pool()`
- **被调用（向上）**: `operator new`、`operator delete`、`operator new[]`、`operator delete[]`
- **知识域**: CRTP 对象池分派

#### pooled_object::operator new()

- **功能说明**: 重载单对象 new 操作符。小对象（<= `policy::max_pool_size`）从目标池分配，大对象直通系统堆。
- **签名**: `void *operator new(std::size_t count)`
- **参数**: `count` — 待分配的字节数
- **返回值**: `void*` — 分配的内存指针
- **调用（向下）**: `get_target_pool()->allocate()` 或 `::operator new()`
- **被调用（向上）**: 继承类的 `new` 表达式
- **知识域**: PMR 对象分配

#### pooled_object::operator delete()

- **功能说明**: 重载单对象 delete 操作符。必须与 `operator new` 对应，归还到正确的位置。
- **签名**: `void operator delete(void *ptr, std::size_t count)`
- **参数**: `ptr` — 待释放的内存指针；`count` — 待释放的字节数
- **返回值**: 无
- **调用（向下）**: `get_target_pool()->deallocate()` 或 `::operator delete()`
- **被调用（向上）**: 继承类的 `delete` 表达式
- **知识域**: PMR 对象释放

#### pooled_object::operator new[]()

- **功能说明**: 重载数组 new 操作符。小对象从目标池分配，大对象直通系统堆。
- **签名**: `void *operator new[](std::size_t count)`
- **参数**: `count` — 待分配的字节数
- **返回值**: `void*` — 分配的内存指针
- **调用（向下）**: `get_target_pool()->allocate()` 或 `::operator new[]()`
- **被调用（向上）**: 继承类的 `new[]` 表达式
- **知识域**: PMR 数组分配

#### pooled_object::operator delete[]()

- **功能说明**: 重载数组 delete 操作符。小对象归还到目标池，大对象直通系统堆。
- **签名**: `void operator delete[](void *ptr, std::size_t count)`
- **参数**: `ptr` — 待释放的内存指针；`count` — 待释放的字节数
- **返回值**: 无
- **调用（向下）**: `get_target_pool()->deallocate()` 或 `::operator delete[]()`
- **被调用（向上）**: 继承类的 `delete[]` 表达式
- **知识域**: PMR 数组释放

### 类: frame_arena

- **功能说明**: 帧分配器（线性分配器）。使用栈上缓冲区和单调增长资源，提供极高的分配性能。适用于短生命周期、高频分配的场景。不可拷贝和移动，分配的内存必须在 `reset()` 或析构前使用。
- **签名**: `class frame_arena`

#### frame_arena::frame_arena()

- **功能说明**: 构造帧分配器。使用 512 字节栈缓冲区和线程局部池作为上游资源，实现无锁性能最大化。内部缓冲覆盖典型 mux 地址头，避免解析时穿透到上游池。
- **签名**: `frame_arena()`
- **参数**: 无
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `system::thread_local_pool()`、`monotonic_buffer` 构造
- **被调用（向上）**: session 级别帧分配器创建
- **知识域**: 帧级线性分配

#### frame_arena::get()

- **功能说明**: 获取内存资源指针。可用于创建 PMR 容器。
- **签名**: `resource_pointer get()`
- **参数**: 无
- **返回值**: `resource_pointer` — 内存资源指针
- **调用（向下）**: 无
- **被调用（向上）**: `recognition::recognize()`、session 级别 PMR 容器创建
- **知识域**: 帧分配器资源导出

#### frame_arena::reset()

- **功能说明**: 重置分配器。释放所有已分配内存，重置游标到初始位置。调用后之前分配的所有内存均失效。
- **签名**: `void reset()`
- **参数**: 无
- **返回值**: 无
- **调用（向下）**: `monotonic_buffer::release()`
- **被调用（向上）**: session 结束或帧切换时
- **知识域**: 帧分配器生命周期管理

---

## memory 概述

Memory 模块的详细设计文档。三级池化架构（global_pool、thread_local_pool、hot_path_pool）配合 PMR 容器别名，为热路径提供零堆分配的内存管理方案。

## 相关页面

- [[channel/connection/pool|pool]] — 连接池，使用 PMR 分配器管理连接对象
- [[agent/config|config]] — 配置中的内存参数
- [[agent/worker/worker|worker]] — worker 线程持有 thread_local_pool
- [[resolve/router|router]] — 路由器使用 current_resource() 分配 DNS 结果
- [[fault/code|fault::code]] — 错误码枚举
