---
title: "ADR 002: PMR 内存策略"
created: 2026-05-23
updated: 2026-05-23
layer: dev
tags: [dev, adr, architecture, memory, pmr]
status: accepted
---

# ADR 002: PMR 内存策略

## 状态

已接受 (Accepted)

## 背景

Prism 作为高性能代理服务器，每个连接的每次数据读写都涉及内存分配。在传统的 `new`/`delete` 内存管理下：

- **堆分配延迟**: 每次 `malloc`/`free` 调用需要获取全局堆锁，延迟在百纳秒到微秒级别
- **内存碎片**: 频繁的小对象分配/释放导致内存碎片，降低缓存命中率
- **多线程竞争**: 多个 worker 线程同时分配内存导致锁争用
- **不可预测的延迟尾部分布**: 堆分配的延迟方差大，P99 延迟不可控

代理服务器的热路径（数据转发循环）每秒执行数百万次，每次多 100 纳秒的堆分配开销会累积成显著的性能损失。

### 方案 A: 对象池 (Object Pool)

预分配固定大小的对象池，按需取用。

- **优势**: 固定大小对象分配极快，O(1)
- **劣势**: 只支持固定大小，不支持变长容器（string、vector）；池大小需要预估；不同对象大小需要多个池

### 方案 B: 自定义分配器 (Custom Allocator)

为 `std::vector`、`std::string` 等容器编写自定义分配器。

- **优势**: 完全控制分配行为
- **劣势**: 需要为每个容器类型编写分配器，维护成本高；与 STL 容器适配复杂

### 方案 C: std::pmr 多态内存资源 (PMR)

使用 C++17 引入的 `std::pmr` 多态内存资源，通过运行时注入内存资源实现零堆分配。

- **优势**: 标准化接口，所有 PMR 容器共享同一内存资源抽象；支持变长分配；可分层组合（单调缓冲区 -> 线程池 -> 全局堆）
- **劣势**: 运行时多态引入间接调用开销（虚函数）；容器类型变长（包含指针）；代码复杂度增加

## 决策

**选择方案 C: std::pmr 多态内存资源 + 帧竞技场**

### 内存层次

Prism 建立了三级内存层次结构：

```
Level 1: frame_arena (帧分配器)
  - 栈上 512 字节缓冲区 + monotonic_buffer_resource
  - 上游: thread_local_pool
  - 用途: 协程帧内临时分配，每次请求结束后 reset()
  - 特点: 分配极快（指针移动），不释放，整体 reset

Level 2: thread_local_pool (线程局部池)
  - unsynchronized_pool_resource（无锁）
  - 上游: new_delete_resource
  - 用途: 热路径对象的长期存储
  - 特点: 无锁，仅限单线程使用

Level 3: global_pool (全局池)
  - synchronized_pool_resource（线程安全）
  - 上游: new_delete_resource
  - 用途: 跨线程传递的对象、启动阶段配置
  - 特点: 线程安全，有互斥锁开销
```

### 容器别名

所有热路径容器使用 PMR 分配器：

```cpp
namespace psm::memory {
    using string = std::pmr::string;                     // PMR 字符串
    using vector<T> = std::pmr::vector<T>;              // PMR 动态数组
    using map<K, V> = std::pmr::map<K, V>;              // PMR 红黑树
    using unordered_map<K, V> = std::pmr::unordered_map; // PMR 哈希表
    using unordered_set<K> = std::pmr::unordered_set;    // PMR 哈希集合
}
```

### 对象池基类

通过 CRTP 模式为热路径对象提供池化分配：

```cpp
template <typename T, pool_type Type = pool_type::local>
class pooled_object {
    void *operator new(size_t count);
    void operator delete(void *ptr, size_t count);
};
```

小对象（<= 16KB）从池分配，大对象直通系统堆。

### 初始化

程序启动时必须调用 `memory::system::enable_global_pooling()` 将默认内存资源设置为全局池：

```cpp
int main() {
    psm::memory::system::enable_global_pooling();
    // ... 后续初始化
}
```

### 策略参数

```cpp
struct policy {
    static constexpr size_t max_blocks = 256;      // 每个 Chunk 最大块数
    static constexpr size_t max_pool_size = 16384;  // 最大池化阈值（16KB）
};
```

### 内存分配性能对比

实际测量中，不同分配策略的延迟差异显著：

| 分配方式 | 平均延迟 | P99 延迟 | 线程安全 |
|----------|----------|----------|----------|
| `new`/`delete` (系统堆) | 50-200ns | 1-10us | 是（有锁） |
| `synchronized_pool_resource` | 20-50ns | 100-500ns | 是（有锁） |
| `unsynchronized_pool_resource` | 5-15ns | 20-50ns | 否（无锁） |
| `monotonic_buffer_resource` | 2-5ns | 5-10ns | 否（无锁） |

在代理服务器每秒百万级分配的场景下，`monotonic_buffer_resource` 相比系统堆可节省数十毫秒的 CPU 时间。

### 实际使用示例

#### 热路径中的帧竞技场

```cpp
// session 处理中的帧分配器
auto session::handle() -> net::awaitable<void> {
    memory::frame_arena arena;  // 栈上 512 字节 + monotonic_buffer

    // 协议解析时使用帧竞技场
    memory::vector<std::byte> header(arena.get());
    // ... 解析完毕后 arena 自动释放

    arena.reset();  // 可选：手动重置复用缓冲区
}
```

#### 连接池的 PMR 容器

```cpp
class connection_pool {
    // PMR unordered_map，使用传入的内存资源
    memory::unordered_map<endpoint_key, memory::vector<idle_item>, endpoint_hash> cache_;

public:
    connection_pool(net::io_context &ioc, memory::resource_pointer resource)
        : cache_(resource) {}  // 所有分配使用传入的资源
};
```

#### 配置结构的 PMR 字符串

```cpp
struct config {
    memory::string host;                        // PMR 字符串
    memory::vector<memory::string> server_names; // PMR vector of PMR string
    memory::map<memory::string, endpoint> reverse_map; // PMR map
};
```

> 参考: [[dev/coding/pmr|PMR 使用规范]]

## 后果

### 优势

1. **热路径零堆分配**:
   - 数据转发循环中的所有容器分配（地址解析头、缓冲区切片等）使用帧竞技场，不进入系统堆
   - 连接池的 `endpoint_key` 查找使用线程局部池，无锁竞争
   - `pooled_object` 基类使 session、duct 等高频对象自动从池分配

2. **缓存友好性**:
   - `monotonic_buffer_resource` 的线性分配模式具有良好的空间局部性
   - 线程局部池避免了全局堆的缓存行争用
   - 池化分配减少了内存碎片，提高 TLB 命中率

3. **延迟可预测性**:
   - 帧竞技场的分配时间是常数级（指针移动）
   - 线程局部池无锁，分配延迟稳定
   - 消除了全局堆锁的尾部延迟（P99 改善显著）

4. **标准化接口**:
   - `std::pmr` 是 C++17 标准的一部分，所有主流编译器支持
   - PMR 容器与标准容器 API 完全兼容，学习成本低
   - 内存资源可以在运行时切换，方便测试和性能分析

### 劣势

1. **代码复杂度增加**:
   - 所有容器类型变为 PMR 版本（`memory::string` 而非 `std::string`）
   - 需要在对象构造时传递内存资源指针
   - 容器间拷贝需要注意内存资源的兼容性
   - 配置结构体全部使用 `memory::string`，JSON 反序列化需要 PMR 感知

2. **运行时多态开销**:
   - 每次分配/释放通过虚函数调用，引入间接跳转
   - 虚函数调用阻止编译器内联优化
   - 在极端高频场景下（每秒千万级分配），虚函数开销可测量

3. **生命周期管理复杂**:
   - `frame_arena` 分配的内存在 `reset()` 后全部失效
   - 从帧竞技场分配的容器不能逃逸出当前请求
   - 线程局部池分配的对象不能跨线程传递
   - `pooled_object` 的 `new`/`delete` 重载可能与 placement new 冲突

4. **调试困难**:
   - PMR 分配器的内存不出现在常规的内存分析工具中
   - 内存泄漏更难追踪（池化内存不立即释放）
   - `monotonic_buffer_resource` 的内存峰值难以精确控制

### 缓解措施

- 使用 `pooled_object` 基类封装分配逻辑，减少手动管理
- 在 `frame_arena` 作用域结束时立即 `reset()`，避免内存膨胀
- 配置结构体使用全局池（跨线程安全），热路径数据使用线程局部池
- 通过 [[dev/testing/stress|压力测试]] 验证内存使用模式
- 通过 [[dev/testing/benchmark|基准测试]] 量化 PMR 的性能收益

## 相关页面

- [[dev/coding/pmr|PMR 使用规范]]
- [[dev/adr/001-pure-coroutine-architecture|ADR 001: 纯协程架构]]
- [[dev/testing/benchmark|基准测试]]
- [[dev/testing/stress|压力测试]]
- [[core/memory/overview|内存管理]]
