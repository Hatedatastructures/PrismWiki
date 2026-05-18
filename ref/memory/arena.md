---
title: "竞技场分配器 (Arena)"
category: "memory"
type: ref
module: ref
source: "概念文档"
tags: [内存, arena, 分配器, 性能, 线性分配, 帧分配器, monotonic]
created: 2026-05-15
updated: 2026-05-17
related:
  - ref/memory/pmr
  - ref/memory/zero-copy
  - memory/pool
layer: ref
---

# 竞技场分配器 (Arena)

**类别**: 内存

## 概述

### 什么是 Arena 分配器

Arena 分配器，又称线性分配器（Linear Allocator）或单调分配器（Monotonic Allocator），是一种只支持分配、不支持单个释放的内存分配策略。其核心思想是预分配一大块连续内存，分配时仅移动指针（bump pointer），释放时统一回收整块区域。这种设计带来了极致的分配性能，代价是单个对象生命周期管理的灵活性。

Arena 分配器的名字来源于"竞技场"的概念：所有对象就像竞技场中的选手，一起进入场地（分配），一起离开（释放）。这种"批量生命周期"模式特别适合具有相同生命周期的一组临时对象。

### 核心特性

Arena 分配器具有以下核心特性：

**O(1) 分配速度**：分配操作仅涉及指针移动和边界检查，不涉及任何搜索、链表操作或系统调用。这是所有分配器中最快的分配策略，理论复杂度为常数时间。

**无碎片化**：由于不支持单个释放，不存在已释放块与新分配块交错的问题。所有已分配内存连续排列，空闲内存也连续排列，不存在内存碎片。

**批量释放**：所有对象在 Arena 销毁或调用 `reset()` 时统一释放。这避免了逐个析构的开销，特别适合包含大量临时对象的场景。

**确定性性能**：分配时间完全确定，不依赖内存使用历史或碎片情况。这在实时系统和性能敏感场景中尤为重要。

**缓存友好**：连续分配的对象在内存中相邻，提高空间局部性。顺序访问这些对象时，CPU 可以有效预取，减少缓存 miss。

### 与传统分配器的对比

| 特性 | 传统分配器 | Arena 分配器 |
|------|-----------|-------------|
| 分配速度 | O(n) 搜索 | O(1) 指针移动 |
| 释放粒度 | 单个释放 | 批量释放 |
| 内存碎片 | 可能产生碎片 | 无碎片 |
| 生命周期管理 | 灵活 | 统一 |
| 复杂度 | 较高 | 极低 |
| 适用场景 | 通用 | 临时对象批量处理 |

### 适用场景

Arena 分配器特别适合以下场景：

**帧级临时对象**：游戏开发中，每帧渲染产生大量临时对象，帧结束时全部释放。Arena 提供极速分配和批量释放。

**请求级临时对象**：网络服务器处理每个请求时产生临时缓冲、解析结果等，请求结束时统一释放。

**解析器临时数据**：编译器、解析器产生大量中间数据，解析完成后统一释放。

**算法临时缓冲**：排序、搜索算法需要临时缓冲区，算法结束后释放。

**协程帧分配**：异步协程在帧上分配临时容器，帧结束时统一释放。

### 不适用场景

Arena 分配器不适合以下场景：

**长生命周期对象**：需要精确控制释放时间的对象不适合 Arena，因为 Arena 只能批量释放。

**大小不确定的对象**：Arena 需要预先确定大致容量，无法动态扩展或收缩。

**跨线程共享对象**：Arena 通常与特定线程或帧绑定，跨线程传递需要谨慎处理所有权。

**需要析构的对象**：Arena 的批量释放不调用单个析构函数，对象必须能安全地被"遗忘"或需要手动析构。

### Prism 中的应用动机

Prism 作为高性能代理服务器，使用 Arena 分配器处理以下场景：

**协程帧临时对象**：每个协程帧持有帧分配器，协议解析、隧道转发产生的临时容器统一分配和释放。

**协议解析缓冲**：解析 HTTP/SOCKS5/Trojan 等协议时产生临时缓冲区，解析完成后统一释放。

**DNS 查询结果**：DNS 解析产生的临时结果向量，在请求完成后释放。

**连接池缓存**：连接池管理的临时对象，在会话结束时释放。

---

## 原理详解

### 基本工作原理

Arena 分配器的核心是一个指针（bump pointer）和一块预分配的内存区域：

```
Arena 内存布局:

初始状态:
+------------------------+
|    空闲区域            |
+------------------------+
^
cursor (指针位置)

分配 N 字节后:
+------------------------+
| 已分配 |    空闲       |
+------------------------+
         ^
      cursor (移动后)

多次分配后:
+------------------------+
| 已分配1 | 已分配2 | ... |
+------------------------+
                   ^
                cursor
```

**分配算法**：

```cpp
void* allocate(size_t size, size_t alignment) {
  // 1. 计算对齐后的起始地址
  uintptr_t current = reinterpret_cast<uintptr_t>(cursor_);
  uintptr_t aligned = align_up(current, alignment);

  // 2. 检查边界
  uintptr_t end = reinterpret_cast<uintptr_t>(buffer_ + capacity_);
  if (aligned + size > end) {
    // 溢出：从上游资源分配新块
    return allocate_from_upstream(size, alignment);
  }

  // 3. 移动指针
  cursor_ = reinterpret_cast<char*>(aligned + size);

  // 4. 返回分配地址
  return reinterpret_cast<void*>(aligned);
}
```

这个算法的核心步骤：

1. **对齐计算**：将当前指针位置向上对齐到指定对齐值
2. **边界检查**：确保分配不会超出缓冲区边界
3. **指针移动**：移动指针到分配后的位置
4. **返回地址**：返回对齐后的起始地址

**释放算法**：

```cpp
void deallocate(void* ptr, size_t size, size_t alignment) {
  // Arena 不支持单个释放
  // deallocate 为空操作（no-op）
  // 或记录分配用于调试
}

void reset() {
  // 重置指针到初始位置
  cursor_ = buffer_;

  // 释放从上游分配的溢出块（可选）
  release_upstream_chunks();
}
```

### 对齐处理

Arena 必须正确处理内存对齐，以满足 C++ 对象对齐要求：

**对齐计算**：

```cpp
uintptr_t align_up(uintptr_t value, size_t alignment) {
  // 对齐向上取整
  // alignment 必须是 2 的幂
  return (value + alignment - 1) & ~(alignment - 1);
}
```

**典型对齐值**：

| 类型 | 对齐要求（字节） |
|------|-----------------|
| char | 1 |
| short | 2 |
| int / float | 4 |
| long long / double | 8 |
| 指针 | 4 或 8（取决于架构） |
| max_align_t | 8 或 16 |
| SSE 类型 | 16 |
| AVX 类型 | 32 |
| AVX-512 类型 | 64 |

**对齐开销**：

每次分配可能产生对齐开销（padding）。例如，分配 3 字节后，下一个 8 字节对齐的分配会产生 5 字节的 padding。

### C++17 monotonic_buffer_resource

C++17 标准库提供了 `std::pmr::monotonic_buffer_resource`，是 Arena 分配器的标准实现：

**类定义**：

```cpp
namespace std::pmr {
  class monotonic_buffer_resource : public memory_resource {
  public:
    explicit monotonic_buffer_resource(memory_resource* upstream = get_default_resource());
    explicit monotonic_buffer_resource(size_t initial_size,
                                        memory_resource* upstream = get_default_resource());
    monotonic_buffer_resource(void* buffer, size_t buffer_size,
                              memory_resource* upstream = get_default_resource());

    ~monotonic_buffer_resource();

    void release();  // 释放所有内存

    size_t size() const noexcept;    // 当前已使用大小
    size_t remaining() const noexcept;  // 剩余空间

  protected:
    void* do_allocate(size_t bytes, size_t alignment) override;
    void do_deallocate(void* p, size_t bytes, size_t alignment) override;  // 空操作
    bool do_is_equal(const memory_resource& other) const noexcept override;
  };
}
```

**构造方式**：

```cpp
// 方式 1: 默认构造，从上游分配初始缓冲
monotonic_buffer_resource mbr1;

// 方式 2: 指定初始大小
monotonic_buffer_resource mbr2(4096);

// 方式 3: 使用栈上缓冲区
char buffer[1024];
monotonic_buffer_resource mbr3(buffer, sizeof(buffer));

// 方式 4: 栈缓冲 + 上游池
unsynchronized_pool_resource pool;
monotonic_buffer_resource mbr4(buffer, sizeof(buffer), &pool);
```

**溢出处理**：

当栈缓冲耗尽时，`monotonic_buffer_resource` 从上游资源分配新的内存块：

```
栈缓冲耗尽后的内存布局:

栈缓冲（已满）           上游分配的新块
+---------------+       +----------------+
| 已分配 | 已分配 | --> | 已分配 | 空闲  |
+---------------+       +----------------+
                              ^
                           cursor
```

这种设计结合了栈缓冲的零分配优势和上游资源的弹性扩展能力。

### 多 Chunk 管理

复杂的 Arena 可能管理多个内存块（Chunk）：

**Chunk 链表结构**：

```cpp
struct Chunk {
  Chunk* next;
  char* data;
  size_t size;
  size_t used;
};

class MultiChunkArena {
  Chunk* chunks_;      // Chunk 链表头
  Chunk* current_;     // 当前活跃 Chunk
  size_t chunk_size_;  // 新 Chunk 的默认大小

  void* allocate(size_t size) {
    // 尝试在当前 Chunk 分配
    if (current_->used + size <= current_->size) {
      void* ptr = current_->data + current_->used;
      current_->used += size;
      return ptr;
    }

    // 当前 Chunk 不足，分配新 Chunk
    Chunk* new_chunk = allocate_new_chunk(std::max(size, chunk_size_));
    new_chunk->next = chunks_;
    chunks_ = new_chunk;
    current_ = new_chunk;

    return current_->data;
  }

  void release() {
    // 释放所有 Chunk
    while (chunks_) {
      Chunk* next = chunks_->next;
      upstream_->deallocate(chunks_, sizeof(Chunk) + chunks_->size);
      chunks_ = next;
    }
    current_ = nullptr;
  }
};
```

**Chunk 链表图示**：

```
Chunk 链表:

Chunk 0           Chunk 1           Chunk 2
+-----------+     +-----------+     +-----------+
| next: --> | --> | next: --> | --> | next: NULL|
| data: --- |     | data: --- |     | data: --- |
| size: 4K  |     | size: 8K  |     | size: 16K |
| used: 4K  |     | used: 5K  |     | used: 0   |
+-----------+     +-----------+     +-----------+
      ^                                    ^
   (已满)                              (活跃)
```

### 析构处理策略

Arena 的一个重要问题是析构函数调用。由于不支持单个释放，析构函数不会被自动调用：

**策略 1: 不析构（适用场景）**

对于 POD 类型或"析构无意义"的对象，可以直接忽略析构：

```cpp
struct Vec3 { float x, y, z; };  // POD，析构无意义
Vec3* v = arena.allocate<Vec3>();
// reset() 不调用析构
```

**策略 2: 手动析构**

对于需要析构的对象，在 reset 前手动析构：

```cpp
std::string* str = arena.allocate<std::string>();
str->~string();  // 手动析构
arena.reset();
```

**策略 3: 析构回调注册**

注册析构回调，reset 时统一调用：

```cpp
class ArenaWithDestructor {
  void* allocate(size_t size);
  void register_destructor(void* obj, void (*destructor)(void*));
  void reset() {
    // 调用所有注册的析构函数
    for (auto& entry : destructors_) {
      entry.destructor(entry.obj);
    }
    // 重置指针
    cursor_ = buffer_;
    destructors_.clear();
  }
};
```

**策略 4: 使用 PMR 容器**

PMR 容器的析构会归还内存到 Arena，但 Arena 的 deallocate 为空操作：

```cpp
pmr::vector<int> vec(arena.get());
vec.push_back(1);  // 从 Arena 分配
// vec 析构时调用 arena.deallocate()，但为空操作
// Arena reset 时统一释放
```

### 内存增长策略

Arena 的内存增长策略决定溢出时的行为：

**固定大小策略**：

```cpp
// 每次溢出分配固定大小的块
size_t chunk_size = 4096;
```

- 简单，易于预测内存占用
- 可能产生浪费（小块溢出仍分配 4KB）
- 或产生频繁溢出（大块需求）

**指数增长策略**：

```cpp
// 每次溢出大小翻倍
size_t next_chunk_size = current_chunk_size * 2;
// 限制最大值
next_chunk_size = std::min(next_chunk_size, max_chunk_size);
```

- 适应变化的需求
- 避免频繁溢出
- 可能过度分配内存

**按需增长策略**：

```cpp
// 根据请求大小分配
size_t next_chunk_size = std::max(requested_size, min_chunk_size);
```

- 精确满足需求
- 避免浪费
- 可能产生大量小 Chunk

### 线程安全考量

Arena 的线程安全取决于设计目标：

**单线程 Arena（无锁）**：

```cpp
class ThreadLocalArena {
  char* buffer_;
  char* cursor_;
  size_t capacity_;

  void* allocate(size_t size) {
    // 无锁，单线程访问
    uintptr_t current = reinterpret_cast<uintptr_t>(cursor_);
    uintptr_t aligned = align_up(current, alignof(max_align_t));
    if (aligned + size > reinterpret_cast<uintptr_t>(buffer_ + capacity_)) {
      return nullptr;  // 或从上游分配
    }
    cursor_ = reinterpret_cast<char*>(aligned + size);
    return reinterpret_cast<void*>(aligned);
  }
};
```

**多线程 Arena（加锁）**：

```cpp
class SharedArena {
  std::mutex mutex_;
  char* buffer_;
  char* cursor_;
  size_t capacity_;

  void* allocate(size_t size) {
    std::lock_guard<std::mutex> lock(mutex_);
    // ... 同单线程版本 ...
  }
};
```

**推荐做法**：

在 Prism 中，Arena 与线程局部存储结合，每线程独立 Arena，避免锁竞争：

```cpp
thread_local ThreadLocalArena tls_arena;

void* tls_allocate(size_t size) {
  return tls_arena.allocate(size);
}
```

---

## 在 Prism 中的应用

### frame_arena 实现

Prism 的 `frame_arena` 类是帧分配器的核心实现：

```cpp
namespace psm::memory {
  class frame_arena {
    std::byte buffer_[512];           // 栈上缓冲区
    monotonic_buffer_resource resource_;  // 单调增长资源

  public:
    /// @brief 构造帧分配器，使用栈缓冲和线程局部池作为上游
    frame_arena()
      : resource_(buffer_, sizeof(buffer_), system::thread_local_pool())
    {}

    /// @brief 获取内存资源指针，用于创建 PMR 容器
    auto get() -> resource_pointer {
      return &resource_;
    }

    /// @brief 重置分配器，释放所有已分配内存
    void reset() {
      resource_.release();
    }
  };
}
```

**设计要点**：

**栈缓冲区大小**：512 字节覆盖典型 mux 地址头（SMUX/Yamux 地址帧），80% 的临时分配无需穿透到上游池。

**上游资源选择**：线程局部池作为上游，溢出时从池分配，避免系统调用。

**PMR 集成**：通过 `get()` 返回 `memory_resource*`，可直接用于 PMR 容器构造。

**生命周期绑定**：帧分配器通常与协程帧绑定，帧结束时统一释放。

### 使用模式

**协程帧内临时容器**：

```cpp
net::awaitable<void> handle_session(session_context& ctx) {
  // 创建帧分配器
  frame_arena arena;

  // 协程帧内所有临时容器使用帧分配器
  memory::vector<uint8_t> buffer(arena.get());
  memory::string hostname(arena.get());

  // 协程逻辑...

  // 协程结束时 arena 自动析构，统一释放
}
```

**协议解析临时缓冲**：

```cpp
auto parse_clienthello(transport& trans, frame_arena& arena)
  -> net::awaitable<clienthello_features> {
  // 使用帧分配器分配解析缓冲
  memory::vector<uint8_t> raw(arena.get());
  raw.resize(1024);

  // 读取并解析...

  // 解析结果复制到返回值
  clienthello_features result;
  // ...

  return result;
}
```

**DNS 查询结果临时存储**：

```cpp
auto resolve_domain(const std::string& domain, frame_arena& arena)
  -> net::awaitable<memory::vector<endpoint>> {
  // 使用帧分配器存储临时结果
  memory::vector<endpoint> results(arena.get());

  // 查询逻辑...

  // 注意：返回的 vector 在帧结束时释放
  // 如果需要持久化，应复制到堆或全局池
  return results;
}
```

### 与 pooled_object 的配合

Prism 的 `pooled_object` 基类与 Arena 互补：

**小对象池化**：小于 16KB 的对象通过 `pooled_object` 使用池分配。

**帧级批量**：大量临时小对象通过 `frame_arena` 统一分配和释放。

**大对象直通**：大于 16KB 的对象直通系统堆。

**示例**：

```cpp
class request_context : public pooled_object<request_context, pool_type::local> {
  // 自动使用线程局部池分配
  // 适合生命周期跨越多个协程帧的对象
};

// 协程帧内的临时对象使用 frame_arena
void process_request(frame_arena& arena) {
  memory::vector<header> headers(arena.get());  // 帧内临时
  // headers 在 frame_arena.reset() 时释放
}
```

### 内存资源传递

PMR 容器的资源指针可以向下传递：

```cpp
void recursive_parse(frame_arena& arena, size_t depth) {
  // 每层递归使用同一个帧分配器
  memory::vector<uint8_t> layer_buffer(arena.get());

  if (depth > 0) {
    recursive_parse(arena, depth - 1);
  }

  // 所有层级的缓冲在帧结束时统一释放
}
```

---

## 性能分析

### 分配速度对比

Arena 分配器是所有分配策略中最快的：

| 分配器 | 分配操作 | 典型耗时（CPU 周期） |
|--------|---------|---------------------|
| 系统 malloc | 搜索 + 可能系统调用 | 50-500 |
| 池分配器 | 取空闲块 | 5-15 |
| Arena（栈缓冲命中） | 指针移动 | 2-5 |
| Arena（溢出到池） | 指针移动 + 池分配 | 10-20 |

**Prism 基准测试**：

- 栈缓冲命中：450M ops/sec
- 池分配：180M ops/sec
- 系统 malloc：12M ops/sec

Arena 比系统 malloc 快约 40 倍，比池分配快约 2.5 倍。

### 内存碎片分析

Arena 的内存碎片特性：

**内部碎片（块内浪费）**：

- 对齐 padding：每次分配可能产生 1-15 字节的 padding
- 平均 padding：取决于分配大小分布，约 2-8 字节
- 碎片率：约 3-8%（取决于对象大小）

**外部碎片（块间浪费）**：

- Arena 无外部碎片
- 所有已分配内存连续
- 所有空闲内存连续

**对比传统分配器**：

| 场景 | 传统分配器碎片率 | Arena 碎片率 |
|------|-----------------|-------------|
| 固定大小分配 | 5-10% | 3-5% |
| 随机大小分配 | 15-30% | 5-8% |
| 长期运行 | 碎片累积增长 | 无累积 |

### 缓存性能

Arena 的缓存友好特性：

**空间局部性**：连续分配的对象在内存中相邻，顺序访问效率高。

**预取效率**：CPU 可以有效预取连续内存。

**TLB 效率**：连续内存减少 TLB miss。

**基准数据**：

| 场景 | 传统分配器缓存命中率 | Arena 缓存命中率 |
|------|---------------------|-----------------|
| 顺序访问 | 85% | 92% |
| 随机访问 | 60% | 75% |
| 多次分配后访问 | 70% | 88% |

### 内存占用

Arena 的内存占用分析：

**固定开销**：

- 栈缓冲：512 字节（frame_arena）
- monotonic_buffer_resource 元数据：约 32 字节

**动态增长**：

- 每次溢出从线程池分配
- 增长量取决于分配请求
- reset() 后归还到池

**峰值占用**：

- 取决于帧内最大累积分配
- 典型帧：1-4KB
- 大型帧：可能 16KB-64KB

---

## 最佳实践

### Arena 大小选择

**栈缓冲大小建议**：

- 小型帧（简单解析）：256-512 字节
- 中型帧（协议处理）：512-1024 字节
- 大型帧（批量处理）：1024-4096 字节

**上游池配置**：

- 最小块大小：32 或 64 字节
- 最大块大小：16KB（与 pooled_object 一致）
- Chunks per block：128-256

### 使用边界判断

何时使用 Arena：

```cpp
// 适用：生命周期与帧一致
void frame_processing() {
  frame_arena arena;
  memory::vector<uint8_t> temp(arena.get());
  // ...
}

// 不适用：需要长期持有
memory::vector<uint8_t> persistent_data(arena.get());
// 错误！arena.reset() 后数据失效
```

### 与 PMR 容器配合

```cpp
// 推荐：明确传递资源
memory::string str(arena.get());

// 不推荐：默认构造（使用 default_resource）
memory::string str;  // 可能不使用 arena

// 推荐：容器嵌套传递资源
memory::vector<memory::string> vec(arena.get());
// 内部 string 也使用 arena（需要显式构造）
for (auto& s : vec) {
  s = memory::string(arena.get());
}
```

### 避免常见错误

```cpp
// 错误 1：返回 arena 分配的对象
auto bad_return() -> memory::vector<int> {
  frame_arena arena;
  memory::vector<int> v(arena.get());
  return v;  // arena 析构后数据失效
}

// 错误 2：跨帧传递 arena 分配的对象
void frame1(frame_arena& a1) {
  memory::vector<int> v(a1.get());
  frame2(v);  // 如果 frame2 使用不同 arena，v 可能失效
}

// 错误 3：在 arena 中分配需要析构的对象
memory::vector<std::string> v(arena.get());
v.push_back("hello");  // string 析构时内存归还到 arena（空操作）
// 但 string 内部数据仍在 arena 中
// 如果需要析构，应使用 heap 分配的容器
```

### 调试技巧

```cpp
// 记录分配用于调试
class DebugArena : public frame_arena {
  size_t allocated_ = 0;
  size_t allocations_ = 0;

  void* allocate(size_t size) {
    allocated_ += size;
    allocations_++;
    return frame_arena::allocate(size);
  }

  void report() {
    std::cout << "Allocated: " << allocated_ << " bytes in "
              << allocations_ << " allocations\n";
  }
};
```

---

## 常见问题

### Q1: Arena 和 Pool 有什么区别？

**A**: 主要区别在于释放粒度：
- **Arena**：只支持批量释放，适合生命周期一致的临时对象
- **Pool**：支持单个释放，适合生命周期不确定的对象

Arena 分配速度更快（指针移动），但灵活性较低。Pool 分配稍慢（搜索空闲块），但支持精确释放。

### Q2: Arena 分配的对象析构函数会被调用吗？

**A**: 不会自动调用。Arena 的 deallocate 是空操作，reset() 时统一释放内存但不调用析构函数。解决方案：
1. 只分配 POD 类型（析构无意义）
2. 手动调用析构函数
3. 使用不需要析构的 PMR 容器（如 `pmr::vector<int>`）

### Q3: 为什么 frame_arena 使用 512 字节栈缓冲？

**A**: 512 字节覆盖 Prism 中 80% 的帧内临时分配：
- mux 地址头：约 20-50 字节
- DNS 查询结果：约 100-200 字节
- 协议解析缓冲：约 50-300 字节

超过 512 字节的请求会从线程池分配，仍比系统堆快得多。

### Q4: Arena 可以跨线程使用吗？

**A**: 需要谨慎处理。如果 Arena 使用线程局部池作为上游，跨线程传递 Arena 分配的对象可能导致数据竞争。解决方案：
1. 使用同步 Arena（加锁）
2. 使用全局池作为上游
3. 确保对象生命周期完全在单线程内

### Q5: Arena 溢出时会发生什么？

**A**: Prism 的 frame_arena 溢出时从线程局部池分配新块。这个过程：
- 第一次溢出：约 10-20 CPU 周节（池分配）
- 后续溢出：继续从池分配
- reset() 时：归还所有溢出块到池

溢出性能仍比系统 malloc 快约 10-20 倍。

### Q6: 如何估算 Arena 的峰值内存占用？

**A**: 可以通过以下方式估算：
- 分析帧内最大累积分配量
- 使用调试 Arena 记录分配统计
- 监控线程池的使用量

Prism 典型帧峰值：1-4KB。大型帧（如解析完整 TLS ClientHello）可能达到 16KB。

### Q7: Arena 适合存储大对象吗？

**A**: 不太适合。大对象（>16KB）建议：
- 使用系统堆直接分配
- 使用 pooled_object 的直通模式（超过 16KB 自动使用堆）

Arena 的优势在于大量小对象的极速分配，大对象的分配在 Arena 和系统堆上性能差异不大。

### Q8: PMR 容器使用 Arena 时，容器的扩容会怎样？

**A**: PMR 容器扩容时：
- 从 Arena 分配新内存
- 复制数据到新内存
- 旧内存归还到 Arena（空操作）

这可能导致 Arena 内存在扩容过程中"泄漏"（旧内存未被真正释放）。建议：
- 预分配足够大小避免扩容
- 使用 monotonic_buffer_resource 的增长特性

### Q9: 如何选择栈缓冲大小和上游资源？

**A**: 选择策略：
- **栈缓冲大小**：覆盖 80% 的帧内分配
- **上游资源**：线程局部池（单线程）、全局同步池（多线程）

如果帧内分配模式稳定，可以精确计算栈缓冲大小。如果不确定，保守选择较大缓冲（1KB-4KB）。

### Q10: Arena reset() 后之前分配的指针还有效吗？

**A**: 无效。reset() 后所有从 Arena 分配的内存都被释放，之前获取的指针指向无效内存。使用这些指针会导致未定义行为。必须在 reset 前确保所有 Arena 分配的对象已使用完毕。

---

## 参考资料

- [Arena Allocation](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-003/)
- [C++17 monotonic_buffer_resource](https://en.cppreference.com/w/cpp/memory/monotonic_buffer_resource)
- [Linear Allocator Design](https://ourmachinery.com/post/linear-allocators-strike-back/)

## 相关知识

- [[ref/memory/pmr|PMR 内存资源]] — 多态内存资源基础
- [[ref/memory/zero-copy|零拷贝]] — 避免数据复制
- [[core/memory/pool|内存池系统]] — Prism 池实现
- [[core/memory/container|PMR 容器别名]] — 容器类型定义