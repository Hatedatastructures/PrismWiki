---
title: PMR 内存池
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [cpp, architecture, networking]
sources:
  - I:/code/Prism/README.md
  - I:/code/Prism/CLAUDE.md
confidence: high
---

# PMR 内存池

PMR（Polymorphic Memory Resource）是 C++17 引入的内存分配器框架，Prism 使用它实现热路径零堆分配。

## 概述

PMR 提供了一种灵活的内存分配策略，允许在运行时选择不同的内存分配器。在 Prism 中，PMR 用于实现全局池和线程独占池，消除热路径上的堆分配。

## 核心概念

### Memory Resource

- **Monotonic Buffer Resource** — 单调递增的缓冲区，只分配不释放
- **Pool Resource** — 内存池，预分配固定大小的块
- **Synchronized Pool Resource** — 线程安全的内存池
- **Unsynchronized Pool Resource** — 非线程安全的内存池（性能更好）

### Allocator

- **Polymorphic Allocator** — 多态分配器，可以使用不同的 Memory Resource
- **Standard Allocator** — 标准分配器，使用全局 new/delete

## 为什么代理引擎需要内存池

代理引擎是典型的高并发 I/O 密集型应用，内存管理面临以下挑战：

### 高频分配/释放

每个代理连接在其生命周期内会产生大量小对象：协议头解析缓冲区、DNS 查询结果、TLS 会话状态、连接元数据等。一个同时处理 10 万连接的引擎，每秒可能产生数百万次内存分配。传统的 `new/delete` 走通用分配器（glibc malloc/jemalloc），每次调用有数十到数百纳秒的开销，累积起来成为显著的性能瓶颈。

### 减少锁竞争

通用分配器通常使用全局锁或分桶锁来管理堆内存。在多线程代理引擎中，多个工作线程同时分配/释放内存会导致严重的锁竞争。PMR 的 `unsynchronized_pool_resource` 为每个线程提供独立的内存池，彻底消除分配器层面的锁竞争。

### 内存碎片

长期运行的代理引擎会经历连接的频繁创建和销毁，导致堆内存碎片化。PMR 的 Pool Resource 使用固定大小的块分配，天然抗碎片。Monotonic Buffer Resource 则完全不释放内存，通过整体重置来回收，从根本上避免碎片问题。

## PMR 的三种资源类型详解

### monotonic_buffer_resource

单调递增缓冲区资源。内存只从预分配的缓冲区前端分配，不单独释放每个分配。当整个资源销毁或调用 `release()` 时，所有内存一次性回收。

适用场景：
- 请求级别的临时对象（协议解析中间结果）
- 生命周期一致的对象组
- 需要极高分配速度但可以批量回收的场景

```cpp
char buffer[65536];  // 栈上预分配
pmr::monotonic_buffer_resource resource(buffer, sizeof(buffer));
pmr::vector<char> header(&resource);  // 从栈缓冲区分配，零堆调用
```

### synchronized_pool_resource

线程安全的内存池资源。内部使用锁保护池的状态，多个线程可以安全地同时分配/释放。

适用场景：
- 全局共享的内存池
- 不同线程间需要共享对象的场景
- 大块内存分配（如大的缓冲区）

### unsynchronized_pool_resource

非线程安全的内存池资源。不使用任何锁，分配/释放速度最快，但只能在单线程中使用。

适用场景：
- 线程独占的内存池（Prism 的 per-worker 模型）
- 热路径上的小对象分配
- 性能敏感的内循环

在 Prism 中，每个工作线程拥有自己的 `unsynchronized_pool_resource`，所有该线程上的连接共享这个池，实现了零锁的内存分配。

## 热路径零分配的实现方式

Prism 的"热路径零分配"策略：

1. **启动时预分配** — 程序启动时，为每个工作线程预分配一大块内存（如 256MB），作为 `monotonic_buffer_resource` 的底层缓冲区
2. **线程独占池** — 每个线程创建 `unsynchronized_pool_resource`，预热常用大小的块（如 64B、256B、1KB、4KB）
3. **连接对象池化** — Connection、Buffer、ProtocolHeader 等高频对象使用 PMR 容器，从线程池分配
4. **请求级临时对象** — 协议解析中的临时对象使用栈上的 `monotonic_buffer_resource`，函数返回即回收
5. **运行时零 malloc** — 热路径上不再调用任何 `new/delete/malloc/free`

```
启动 → 预分配内存块 → 创建线程池资源 → 预热块大小
运行 → 连接创建（从池分配）→ 数据处理（从栈缓冲区分配）→ 连接销毁（归还池）
```

## 内存池 vs 普通 new/delete 的性能对比

| 指标 | PMR unsynchronized_pool | 普通 new/delete (glibc) | jemalloc |
|------|------------------------|------------------------|----------|
| 单次分配延迟 | ~5-15ns | ~50-200ns | ~30-100ns |
| 单次释放延迟 | ~5-10ns | ~30-150ns | ~20-80ns |
| 锁竞争 | 无（线程独占） | 有（arena 锁） | 较少（多 arena） |
| 内存碎片 | 极低（固定块） | 中等 | 较低 |
| 额外元数据 | 无（池管理） | 16-32B/分配 | 8-16B/分配 |
| 适用场景 | 高频小对象 | 通用 | 通用 |

在代理引擎的热路径上，PMR 内存池的分配速度比普通分配器快 5-10 倍，且完全消除锁竞争。

## 在 Prism 中的应用

Prism 使用 PMR 实现：

1. **全局池** — 所有线程共享的内存池，用于大块内存分配
2. **线程独占池** — 每个线程独立的内存池，避免锁竞争
3. **热路径零分配** — 预分配所有需要的内存，运行时不再调用 new/delete
4. **零拷贝** — 数据直接在内存池间传递，无需复制

## 示例代码

```cpp
// 创建内存池
pmr::synchronized_pool_resource pool;

// 使用内存池分配
pmr::polymorphic_allocator<int> alloc(&pool);
int* p = alloc.allocate(1);

// 在 Prism 中的应用
class Connection {
    pmr::polymorphic_allocator<char> alloc_;
    pmr::vector<char> buffer_;

public:
    Connection(pmr::memory_resource* resource)
        : alloc_(resource), buffer_(resource) {}
};
```

## 优势

- **性能** — 减少堆分配，提高分配速度
- **可预测** — 内存分配模式可预测，减少碎片
- **灵活** — 可以根据场景选择不同的分配策略
- **线程安全** — 可以选择线程安全或非线程安全的实现

## 相关页面

- [[agent]] — 使用 PMR 的服务端代理引擎
- [[cpp23-coroutines]] — 协程，协程帧也从 PMR 池分配
- [[dev/tcp]] — 传输层协议，每个 TCP 连接的缓冲区使用 PMR
- [[docs/protocol/proxy-protocols]] — 代理协议，协议解析对象使用 PMR 分配
- [[Mihomo-Clash-Config]] — mihomo 客户端配置参考
