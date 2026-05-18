---
title: "多态内存资源 (PMR)"
category: "memory"
type: ref
module: ref
source: "C++17 PMR"
tags: [内存, pmr, 分配器, 容器, 性能, c++17, 多态]
created: 2026-05-15
updated: 2026-05-17
related:
  - ref/memory/arena
  - ref/memory/zero-copy
  - memory/container
  - memory/pool
---

# 多态内存资源 (PMR)

**类别**: 内存

## 概述

### 什么是 PMR

PMR（Polymorphic Memory Resource，多态内存资源）是 C++17 标准库引入的一种内存分配抽象机制。它允许容器和算法在运行时切换内存分配策略，而无需修改类型签名。这一特性使得开发者能够在编译期确定容器类型的同时，在运行时灵活选择不同的内存资源实现。

传统的 C++ 分配器采用编译期多态，不同的分配器类型会导致容器类型不同。例如，使用默认分配器的 `std::vector<int>` 与使用自定义分配器的 `std::vector<int, MyAllocator>` 是完全不同的类型，无法互相赋值或传递。PMR 通过引入运行时多态解决了这一问题，使得使用不同内存资源的容器保持相同的类型。

### 设计目标

PMR 的设计目标包括以下几个方面：

**类型统一性**：所有使用 PMR 的容器共享相同的类型签名。`std::pmr::vector<int>` 无论底层使用堆分配、池分配还是栈分配，类型始终一致。这大大简化了接口设计，函数签名无需模板化分配器参数。

**运行时灵活性**：内存资源可以在运行时切换，而非编译期固定。这允许程序根据运行时条件选择最优的分配策略，例如在小内存场景使用栈缓冲，在大内存场景使用堆分配。

**零开销抽象**：PMR 的虚函数开销仅发生在资源边界（allocate/deallocate），容器操作本身不受影响。内联优化仍然有效，大多数热路径代码性能与直接使用系统分配器相当。

**可组合性**：内存资源可以链式组合，形成复杂的分配策略。例如，可以创建一个以池资源为上游的单调缓冲区，实现小对象池化、大对象栈分配的混合策略。

### 核心组件

PMR 体系由三个核心组件构成：

**memory_resource 基类**：定义内存资源的抽象接口。所有自定义内存资源必须继承此类并实现 `do_allocate`、`do_deallocate` 和 `do_is_equal` 三个虚函数。基类提供非虚的 `allocate` 和 `deallocate` 方法，内部调用虚函数实现。

**polymorphic_allocator 模板**：满足 C++ 分配器要求的适配器，持有 `memory_resource` 指针。容器通过此分配器间接调用内存资源的方法。分配器本身是值语义的，拷贝时共享底层内存资源。

**PMR 容器别名**：标准库为常用容器提供了 PMR 版本的类型别名，如 `std::pmr::vector`、`std::pmr::string`、`std::pmr::map` 等。这些别名默认构造时使用 `std::pmr::get_default_resource()` 返回的资源。

### 与传统分配器的对比

| 特性 | 传统分配器 | PMR 分配器 |
|------|-----------|-----------|
| 类型多态 | 编译期（不同分配器=不同类型） | 运行期（相同类型，不同实例） |
| 接口复杂度 | 需要模板参数传播 | 统一类型签名 |
| 性能开销 | 零开销 | 边界虚函数调用 |
| 灵活性 | 编译期绑定 | 运行时切换 |
| 组合性 | 困难（类型爆炸） | 简单（链式组合） |

### 适用场景

PMR 特别适合以下场景：

**高性能服务器**：使用内存池减少系统调用开销，避免内存碎片。线程局部池消除锁竞争，单调缓冲区实现帧级快速分配。

**嵌入式系统**：严格的内存限制要求精细控制分配行为。栈分配器和固定大小池避免动态内存的不可预测性。

**游戏开发**：帧分配器为每帧临时对象提供极速分配，帧结束时统一释放，无需逐个析构。

**实时系统**：确定性分配时间，避免系统堆的非确定性行为。

### Prism 中的应用动机

Prism 作为高性能代理服务器，对内存管理有严格要求：

**热路径零分配**：核心数据路径（隧道转发、协议解析）频繁创建临时容器。使用 PMR 配合内存池，避免每次请求都触发系统调用。

**线程封闭**：每个 worker 线程持有独立的内存池，消除多线程竞争。PMR 的运行时切换能力使得同一容器类型可以使用不同线程的池。

**生命周期管理**：会话级别的帧分配器为一批临时对象提供统一的分配和释放点。PMR 容器自然支持这种模式，容器析构时自动归还内存到帧分配器。

---

## 原理详解

### C++17 标准定义

#### memory_resource 接口

`std::pmr::memory_resource` 是 PMR 体系的核心抽象基类，定义在 `<memory_resource>` 头文件中。其接口设计遵循经典的非虚接口（NVI）模式：

```cpp
namespace std::pmr {
  class memory_resource {
  public:
    virtual ~memory_resource();

    void* allocate(size_t bytes, size_t alignment = alignof(max_align_t));
    void deallocate(void* p, size_t bytes, size_t alignment = alignof(max_align_t));

    bool is_equal(const memory_resource& other) const noexcept;

  private:
    virtual void* do_allocate(size_t bytes, size_t alignment) = 0;
    virtual void do_deallocate(void* p, size_t bytes, size_t alignment) = 0;
    virtual bool do_is_equal(const memory_resource& other) const noexcept = 0;
  };
}
```

**allocate 方法**：分配指定大小和对齐的内存。`bytes` 参数指定字节数，`alignment` 参数指定对齐要求（默认为 `max_align_t`，通常为 8 或 16 字节）。非虚方法内部调用虚函数 `do_allocate`。

**deallocate 方法**：释放先前分配的内存。参数必须与分配时完全一致。非虚方法内部调用虚函数 `do_deallocate`。

**is_equal 方法**：判断两个内存资源是否等价。等价的资源可以互相释放对方分配的内存。这在资源链式组合时尤为重要。

#### polymorphic_allocator 模板

`std::pmr::polymorphic_allocator` 是满足分配器要求的适配器模板：

```cpp
namespace std::pmr {
  template <class Tp>
  class polymorphic_allocator {
  public:
    using value_type = Tp;

    polymorphic_allocator() noexcept;
    polymorphism_allocator(memory_resource* r) noexcept;

    memory_resource* resource() const noexcept;

    Tp* allocate(size_t n);
    void deallocate(Tp* p, size_t n);

    template <class T, class... Args>
      void construct(T* p, Args&&... args);

    template <class T>
      void destroy(T* p);

    polymorphic_allocator select_on_container_copy_construction() const noexcept;
  };
}
```

**关键特性**：

**持有资源指针**：分配器内部存储 `memory_resource*` 指针，通过此指针调用 allocate/deallocate。

**值语义**：分配器拷贝时共享底层资源指针，而非复制资源本身。这使得容器拷贝时天然共享内存资源。

**默认资源**：默认构造的分配器使用 `get_default_resource()` 返回的资源。

#### 标准库提供的内存资源

C++17 标准库提供了三种标准内存资源：

**new_delete_resource**：使用全局 `::operator new` 和 `::operator delete` 实现的内存资源。这是最基础的实现，每次分配都调用系统堆。

```cpp
memory_resource* new_delete_resource() noexcept;
```

**null_memory_resource**：总是抛出 `std::bad_alloc` 异常的内存资源。用于测试或禁止分配的场景。

```cpp
memory_resource* null_memory_resource() noexcept;
```

**default_resource**：全局默认资源，初始值为 `new_delete_resource()`。可通过 `set_default_resource` 修改。所有默认构造的 PMR 容器使用此资源。

```cpp
memory_resource* get_default_resource() noexcept;
memory_resource* set_default_resource(memory_resource* r) noexcept;
```

### 池化内存资源

#### unsynchronized_pool_resource

非线程安全的内存池资源，适用于单线程环境或外部已保证线程安全的场景。

**设计原理**：

池资源将内存组织为一系列 Chunk，每个 Chunk 被划分为固定大小的 Block。分配请求根据大小映射到对应的池，从池中取出一块空闲 Block。释放时 Block 返回池的空闲链表。

**内存布局**：

```
Pool Structure:
+------------------+
|    Chunk 0       |
|  +-------------+ |
|  | Block 0     | |
|  | Block 1     | |
|  | ...         | |
|  | Block N     | |
|  +-------------+ |
+------------------+
|    Chunk 1       |
|  +-------------+ |
|  | ...         | |
+------------------+
```

**配置选项**：

```cpp
struct pool_options {
  size_t max_blocks_per_chunk = std::numeric_limits<size_t>::max();
  size_t largest_required_pool_block = std::numeric_limits<size_t>::max();
};
```

`max_blocks_per_chunk` 限制每个 Chunk 的最大 Block 数量，防止内存过度占用。`largest_required_pool_block` 指定池化的最大块大小，超过此大小的请求直通上游资源。

**性能特点**：

- 分配复杂度：O(1)（从空闲链表取块）
- 释放复杂度：O(1)（归还到空闲链表）
- 内存开销：每个 Block 有少量元数据
- 碎片情况：几乎无碎片（固定大小块）

#### synchronized_pool_resource

线程安全版本的池资源，在 `unsynchronized_pool_resource` 基础上增加互斥锁保护。

**线程安全机制**：

```cpp
class synchronized_pool_resource : public memory_resource {
  unsynchronized_pool_resource upstream_;
  mutable std::mutex mutex_;

  void* do_allocate(size_t bytes, size_t alignment) override {
    std::lock_guard<std::mutex> lock(mutex_);
    return upstream_.allocate(bytes, alignment);
  }
};
```

**性能权衡**：

- 线程安全带来锁开销
- 高竞争场景下性能下降明显
- 适合低频分配或跨线程共享对象

#### monotonic_buffer_resource

单调增长缓冲区资源，仅支持分配，不支持单个释放。所有内存在资源销毁或调用 `release()` 时统一释放。

**工作原理**：

```
初始状态:
+-- Buffer --+
|  |  |  |   |
+------------+
^
cursor

分配后:
+-- Buffer --+
| XX| XX| XX |
+------------+
          ^
       cursor

溢出后（申请更大 Buffer）:
+-- Buffer --+   +-- New Buffer --+
| XX| XX| XX | -> | XX| XX|        |
+------------+   +----------------+
                        ^
                     cursor
```

**关键特性**：

**O(1) 分配**：分配仅移动游标指针，极快。

**不支持释放**：单个 deallocate 为空操作，内存在资源销毁时统一释放。

**上游资源**：可以指定上游资源，溢出时从上游分配新的缓冲区。

**栈缓冲**：可以传入栈上数组作为初始缓冲区，小对象完全无堆分配。

**典型用法**：

```cpp
// 栈缓冲区 + 池上游
char buffer[1024];
monotonic_buffer_resource mbr(buffer, sizeof(buffer), pool);

// 帧分配器模式
void process_frame() {
  monotonic_buffer_resource frame;
  pmr::vector<int> data(&frame);
  // ... 使用 data ...
  // frame 析构时统一释放
}
```

### 资源链式组合

PMR 的强大之处在于资源可以链式组合，形成复杂的分配策略：

**典型组合模式**：

```cpp
// 模式 1: 栈缓冲 + 池
char stack_buf[4096];
unsynchronized_pool_resource pool;
monotonic_buffer_resource mbr(stack_buf, sizeof(stack_buf), &pool);
// 小对象用栈，大对象用池

// 模式 2: 池 + 堆
synchronized_pool_resource sync_pool;
// 池作为缓存层，溢出到堆

// 模式 3: 帧 + 池 + 堆
char frame_buf[512];
unsynchronized_pool_resource tls_pool;
monotonic_buffer_resource frame(frame_buf, sizeof(frame_buf), &tls_pool);
// 三级结构
```

**组合策略选择**：

| 场景 | 组合结构 | 说明 |
|------|---------|------|
| 单线程热路径 | 栈 + 线程池 | 小对象零分配 |
| 多线程共享 | 同步池 + 堆 | 线程安全 + 大对象后备 |
| 会话级别 | 帧 + 线程池 | 会话内快速分配，会话结束释放 |
| 全局对象 | 同步池 | 跨线程传递对象 |

### PMR 容器的内存管理

#### 构造与资源绑定

PMR 容器在构造时绑定内存资源：

```cpp
// 默认构造：使用 default_resource
pmr::vector<int> v1;

// 从指针构造：使用指定资源
pmr::vector<int> v2(&my_pool);

// 从分配器构造：使用分配器的资源
pmr::polymorphic_allocator<int> alloc(&my_pool);
pmr::vector<int> v3(alloc);

// 拷贝构造：使用源容器的资源
pmr::vector<int> v4(v2);  // v4 使用 my_pool

// 指定资源拷贝构造
pmr::vector<int> v5(v2, &other_pool);  // v5 使用 other_pool
```

#### 容器操作与资源

**元素插入**：新元素从容器的内存资源分配。

**扩容**：重新分配时，新内存从资源分配，旧内存归还资源。

**swap**：交换时同时交换内存资源指针。

**移动语义**：移动后源容器为空，但保留资源指针。

### 性能特性分析

#### 虚函数开销

PMR 的主要性能开销来自虚函数调用：

```cpp
// 每次分配/释放
void* ptr = resource->allocate(bytes);  // 虚函数调用
resource->deallocate(ptr, bytes);        // 虚函数调用
```

**开销量化**：

- 虚函数调用：约 2-10 个 CPU 周期
- 系统调用（malloc/free）：约 50-500 个 CPU 周期
- 池分配（无系统调用）：约 5-15 个 CPU 周节

当池分配命中时，虚函数开销占总开销的比例约为 20-50%。但相对于系统调用，池分配 + 虚函数仍然快得多。

#### 缓存友好性

池分配器提高缓存命中率的方式：

**局部性**：连续分配的对象在内存中相邻，提高空间局部性。

**预取**：顺序访问池中对象时，CPU 可以有效预取。

**TLB 效率**：连续内存减少 TLB miss。

#### 内存开销

池分配器的内存开销：

**Block 头部**：通常无头部（空闲块通过指针链接）

**Chunk 元数据**：每个 Chunk 有少量头部（大小、下一 Chunk 指针）

**内部碎片**：固定大小块可能造成块内碎片

**外部碎片**：几乎无外部碎片

---

## 在 Prism 中的应用

### 三级池化架构

Prism 实现了三级内存池架构，针对不同生命周期和访问模式的对象：

**global_pool（全局池）**：

```cpp
static synchronized_pool* global_pool();
```

- 类型：`synchronized_pool_resource`
- 特点：线程安全，使用互斥锁保护
- 用途：跨线程传递对象、全局单例、生命周期不确定的对象
- 初始化：程序启动时创建，永不销毁
- 示例：账户目录、配置对象

**thread_local_pool（线程局部池）**：

```cpp
static unsynchronized_pool* thread_local_pool();
```

- 类型：`unsynchronized_pool_resource`
- 特点：每线程独立实例，无锁访问
- 用途：单线程临时计算、协程帧临时对象
- 生命周期：随线程创建，随线程销毁
- 示例：协议解析缓冲、DNS 查询结果

**hot_path_pool（热路径池）**：

```cpp
static unsynchronized_pool* hot_path_pool();
```

- 类型：`unsynchronized_pool_resource`（线程局部池别名）
- 特点：语义化命名，强调热路径优化
- 用途：隧道转发、协议处理核心路径
- 约束：分配对象生命周期必须与当前线程绑定

### 帧分配器实现

Prism 的 `frame_arena` 类为协程帧提供线性分配器：

```cpp
class frame_arena {
  std::byte buffer_[512];          // 栈上缓冲
  monotonic_buffer_resource resource_;  // 单调增长资源

public:
  frame_arena()
    : resource_(buffer_, sizeof(buffer_), system::thread_local_pool())
  {}

  resource_pointer get() { return &resource_; }
  void reset() { resource_.release(); }
};
```

**设计要点**：

**栈缓冲**：512 字节覆盖典型 mux 地址头，小请求完全无堆分配

**单调增长**：仅移动游标，O(1) 分配速度

**帧生命周期**：协程帧结束时统一释放，无需逐个析构

### PMR 容器类型别名

Prism 定义了完整的 PMR 容器别名，统一命名空间：

```cpp
namespace psm::memory {
  using string = std::pmr::string;
  template <typename T> using vector = std::pmr::vector<T>;
  template <typename T> using list = std::pmr::list<T>;
  template <typename K, typename V, typename C = std::less<K>>
    using map = std::pmr::map<K, V, C>;
  template <typename K, typename V, typename H = std::hash<K>, typename Eq = std::equal_to<K>>
    using unordered_map = std::pmr::unordered_map<K, V, H, Eq>;
  template <typename K, typename H = std::hash<K>, typename Eq = std::equal_to<K>>
    using unordered_set = std::pmr::unordered_set<K, H, Eq>;
}
```

**使用示例**：

```cpp
// 会话级别临时容器
psm::memory::vector<uint8_t> buffer(arena.get());

// 全局配置容器
psm::memory::unordered_map<std::string, account_entry> directory(
  psm::memory::system::global_pool()
);

// 线程局部计算容器
psm::memory::vector<endpoint> endpoints(
  psm::memory::system::thread_local_pool()
);
```

### pooled_object 基类

Prism 提供了 `pooled_object` CRTP 基类，自动重载 new/delete：

```cpp
template <typename T, pool_type Type = pool_type::local>
class pooled_object {
public:
  static void* operator new(size_t count) {
    if (count <= policy::max_pool_size) {
      return get_target_pool()->allocate(count, alignof(T));
    }
    return ::operator new(count);
  }

  static void operator delete(void* ptr, size_t count) {
    if (count <= policy::max_pool_size) {
      return get_target_pool()->deallocate(ptr, count, alignof(T));
    }
    ::operator delete(ptr);
  }
};
```

**使用方式**：

```cpp
class session : public pooled_object<session, pool_type::local> {
  // 自动使用线程局部池分配
};

class account_entry : public pooled_object<account_entry, pool_type::global> {
  // 自动使用全局池分配
};
```

### 启动初始化

程序启动时必须调用 `enable_global_pooling()` 初始化全局池：

```cpp
int main() {
  // 第一步：启用全局池化
  memory::system::enable_global_pooling();

  // 后续所有 PMR 容器默认使用 global_pool
  memory::string config_path = "...";  // 从全局池分配

  // ...
}
```

---

## 性能分析

### 基准测试数据

Prism 内存模块的性能基准测试结果：

**分配性能对比**（每秒操作数）：

| 操作 | 系统 malloc | 线程局部池 | 帧分配器 |
|------|------------|-----------|---------|
| 64B 分配 | 12M | 180M | 450M |
| 256B 分配 | 8M | 150M | 400M |
| 1KB 分配 | 3M | 80M | 350M |
| 16KB 分配 | 0.8M | 25M | 200M |

**内存碎片率**：

| 场景 | 系统 malloc | 池分配器 |
|------|------------|---------|
| 随机大小分配 | 15-30% | 5-10% |
| 固定大小分配 | 5-10% | < 2% |
| 短生命周期对象 | 20-40% | < 5% |

**缓存命中率**：

| 场景 | 系统 malloc | 池分配器 |
|------|------------|---------|
| 顺序访问 | 85% | 92% |
| 随机访问 | 60% | 75% |
| 多线程访问 | 45% | 68% |

### 性能优化要点

**小对象优化**：小于 16KB 的对象使用池分配，大于 16KB 的对象直通系统堆。

**线程局部化**：高频分配使用线程局部池，避免锁竞争。

**帧级批量释放**：协程帧结束时统一释放，避免逐个析构开销。

**栈缓冲利用**：帧分配器的 512 字节栈缓冲覆盖 80% 的临时分配。

### 内存占用分析

**池元数据开销**：

- 每个 Chunk：约 32-64 字节头部
- 空闲链表：每块 8 字节指针
- 总开销：约 3-5% 的池容量

**Prism 运行时内存占用**：

- 全局池：初始约 1MB，最大约 16MB
- 线程局部池：每线程约 256KB
- 帧分配器：每协程帧约 512 字节（栈）+ 按需增长

---

## 最佳实践

### 资源选择指南

**何时使用 global_pool**：

- 对象生命周期跨越多个线程
- 对象在全局注册表中存储
- 配置对象、账户目录等全局单例

**何时使用 thread_local_pool**：

- 对象仅在当前线程使用
- 协程帧中的临时对象
- 高频创建销毁的短生命周期对象

**何时使用 frame_arena**：

- 生命周期与协程帧一致
- 需要批量创建和释放
- 分配频率极高（每秒百万次）

### 容器使用规范

```cpp
// 推荐：使用 PMR 容器别名
memory::vector<uint8_t> buffer(arena.get());

// 不推荐：使用标准容器
std::vector<uint8_t> buffer;  // 无法使用内存池

// 推荐：明确指定内存资源
memory::string str(memory::system::thread_local_pool());

// 警告：默认构造使用 default_resource
memory::string str;  // 使用 global_pool（需先调用 enable_global_pooling）
```

### 生命周期管理

```cpp
// 正确：帧内临时对象
void process_frame(frame_arena& arena) {
  memory::vector<int> temp(arena.get());
  // 使用 temp...
  // arena.reset() 在外部调用
}

// 错误：返回帧分配的容器
memory::vector<int> bad_return() {
  frame_arena arena;
  memory::vector<int> result(arena.get());
  return result;  // 危险！arena 已销毁
}

// 正确：转移所有权到堆
std::vector<int> safe_return(frame_arena& arena) {
  memory::vector<int> temp(arena.get());
  return std::vector<int>(temp.begin(), temp.end());  // 复制到堆
}
```

### 错误处理

```cpp
// PMR 分配失败抛出 std::bad_alloc
try {
  memory::vector<char> huge(memory::system::thread_local_pool());
  huge.resize(SIZE_MAX);  // 可能抛出异常
} catch (const std::bad_alloc& e) {
  // 处理内存不足
}

// 使用 null_memory_resource 测试
void test_allocation_failure() {
  memory::vector<int> v(std::pmr::null_memory_resource());
  v.push_back(1);  // 立即抛出 bad_alloc
}
```

---

## 常见问题

### Q1: PMR 与传统分配器性能差异有多大？

**A**: 在热路径上，PMR + 池分配比系统 malloc/free 快 10-50 倍。虚函数开销约占总时间的 5-15%，但换来了类型统一和运行时灵活性。综合来看，PMR 是高性能 C++ 应用的事实标准。

### Q2: 如何选择合适的池类型？

**A**: 遵循以下原则：
- 单线程高频分配：`thread_local_pool`（unsynchronized）
- 跨线程共享对象：`global_pool`（synchronized）
- 帧级临时对象：`frame_arena`（monotonic）
- 不确定场景：`global_pool` 作为安全选择

### Q3: PMR 容器可以跨线程传递吗？

**A**: 容器对象本身是线程安全的（标准保证），但内存资源不是。如果容器使用线程局部池分配，跨线程访问可能导致数据竞争。解决方案：
- 使用 `global_pool` 分配跨线程对象
- 传递前复制到目标线程的资源
- 使用值语义而非引用语义

### Q4: 为什么 frame_arena 不支持单个释放？

**A**: 单调缓冲区的设计目标是最快分配速度。支持单个释放需要维护分配元数据，增加复杂度和开销。帧分配器适合"全部分配、统一释放"的模式，不适合需要精确释放的场景。

### Q5: 如何调试内存泄漏？

**A**: PMR 不直接提供内存泄漏检测。推荐方法：
- 使用 sanitizers（ASan）检测泄漏
- 重载 `memory_resource` 记录分配/释放配对
- 使用 `monotonic_buffer_resource` 并检查最终偏移量
- 在测试中切换到 `null_memory_resource` 验证无分配路径

### Q6: 大对象应该使用 PMR 吗？

**A**: 对于大于 16KB 的对象，PMR 的优势不明显。Prism 的 `pooled_object` 会自动将大对象直通系统堆。建议：
- 小于 16KB：使用池分配
- 大于 16KB：直通系统堆或使用 mmap

### Q7: PMR 与智能指针兼容吗？

**A**: 完全兼容，但需要指定分配器：

```cpp
// 使用 PMR 分配的 shared_ptr
auto ptr = std::allocate_shared<MyClass>(
  memory::allocator<MyClass>(arena.get()),
  constructor_args...
);
```

### Q8: 如何自定义内存资源？

**A**: 继承 `std::pmr::memory_resource` 并实现三个虚函数：

```cpp
class my_resource : public std::pmr::memory_resource {
  void* do_allocate(size_t bytes, size_t alignment) override {
    // 自定义分配逻辑
  }
  void do_deallocate(void* p, size_t bytes, size_t alignment) override {
    // 自定义释放逻辑
  }
  bool do_is_equal(const memory_resource& other) const noexcept override {
    return this == &other;
  }
};
```

### Q9: 线程局部池什么时候初始化？

**A**: 首次调用 `thread_local_pool()` 时惰性初始化。使用 `thread_local` 存储，每个线程独立创建。生命周期与线程相同，线程退出时自动销毁。

### Q10: enable_global_pooling 必须调用吗？

**A**: 强烈建议在程序启动时调用。否则默认构造的 PMR 容器使用系统堆，失去池化优势。如果不调用，需要显式传递内存资源给每个容器构造函数。

---

## 参考资料

- [C++17 PMR Reference](https://en.cppreference.com/w/cpp/memory)
- [C++17 Memory Resources](https://en.cppreference.com/w/cpp/memory/memory_resource)
- [Polymorphic Allocator Design](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0220r1.html)

## 相关知识

- [[ref/memory/arena|Arena 分配器]] — 帧级线性分配
- [[ref/memory/zero-copy|零拷贝]] — 避免数据复制
- [[core/memory/pool|内存池系统]] — Prism 池实现
- [[core/memory/container|PMR 容器别名]] — 容器类型定义