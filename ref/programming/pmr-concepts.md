---
title: PMR Concepts
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [programming, cpp, pmr, memory, allocator]
---

# PMR 概念

> Polymorphic Memory Resources（PMR）原理、使用场景、Prism 应用。
> 最后更新：2026-05-17

---

## PMR 概念

### 传统 Allocator 问题

```cpp
// 传统 allocator
std::vector<int, MyAllocator<int>> vec1;
std::vector<int, MyAllocator<int>> vec2;

// 问题：allocator 类型不同无法赋值
// std::vector<int, OtherAllocator<int>> vec3;
// vec1 = vec3;  // 错误！类型不同
```

### PMR 解决方案

PMR 使用运行期多态：

```cpp
// PMR vector
std::pmr::vector<int> vec1(&pool);
std::pmr::vector<int> vec2(&pool);

// 不同 allocator 实例，相同类型
std::pmr::monotonic_buffer_resource pool2;
std::pmr::vector<int> vec3(&pool2);
vec1 = vec3;  // OK！类型相同
```

---

## memory_resource

### 抽象基类

```cpp
class memory_resource {
public:
    virtual void* allocate(size_t bytes, size_t alignment) = 0;
    virtual void deallocate(void* p, size_t bytes, size_t alignment) = 0;
    virtual bool is_equal(const memory_resource& other) const noexcept = 0;
};
```

### 标准实现

| 资源类型 | 特点 | 用途 |
|----------|------|------|
| `new_delete_resource` | 默认，堆分配 | 通用 |
| `null_memory_resource` | 分配失败 | 测试 |
| `monotonic_buffer_resource` | 单向增长，无释放 | 临时缓冲 |
| `pool_resource` | 池化分配 | 高性能 |
| `unsynchronized_pool_resource` | 无锁池 | 单线程 |
| `synchronized_pool_resource` | 锁保护池 | 多线程 |

---

## PMR 容器

### 类型

```cpp
namespace std::pmr {
    using vector = std::vector<T, polymorphic_allocator<T>>;
    using string = std::basic_string<char, std::char_traits<char>, polymorphic_allocator<char>>;
    using deque = std::deque<T, polymorphic_allocator<T>>;
    using list = std::list<T, polymorphic_allocator<T>>;
    // ...
}
```

### 使用

```cpp
// 创建 PMR 资源
std::pmr::monotonic_buffer_resource pool;

// 使用 PMR 容器
std::pmr::vector<int> vec(&pool);
std::pmr::string str(&pool);
std::pmr::deque<int> dq(&pool);

// 容器之间可以赋值
std::pmr::vector<int> vec2(&other_pool);
vec = vec2;  // OK
```

---

## monotonic_buffer_resource

### 特点

- 只增长，不释放（直到 destroy）
- 极快分配（指针前进）
- 无碎片
- 适合临时、短生命周期

### 使用

```cpp
// 固定大小缓冲
std::array<uint8_t, 64_KB> buffer;
std::pmr::monotonic_buffer_resource pool(buffer.data(), buffer.size());

// 或自动增长
std::pmr::monotonic_buffer_resource pool;  // 无限增长

// 使用
std::pmr::vector<uint8_t> data(&pool);
data.resize(1024);  // 从 pool 分配

// pool.destroy();  // 一次性释放所有
```

### Prism 应用

```cpp
// 协程帧内临时缓冲
struct SessionContext {
    std::array<uint8_t, 128_KB> buffer_;
    std::pmr::monotonic_buffer_resource pool_;
    
    std::pmr::vector<uint8_t> request_data{&pool_};
    std::pmr::string request_str{&pool_};
};
```

详见 [[core/memory/overview|Memory 模块]]。

---

## pool_resource

### synchronized_pool_resource

多线程安全池：

```cpp
std::pmr::synchronized_pool_resource pool;

// 多线程安全
std::pmr::vector<int> vec(&pool);
```

### unsynchronized_pool_resource

单线程高性能池：

```cpp
std::pmr::unsynchronized_pool_resource pool;

// 单线程使用
std::pmr::vector<int> vec(&pool);
```

### 池配置

```cpp
// 配置块大小
std::pmr::pool_options opt;
opt.max_blocks_per_chunk = 1024;
opt.largest_required_pool_block = 1024;

std::pmr::synchronized_pool_resource pool(opt);
```

---

## polymorphic_allocator

### 用法

```cpp
// 从 memory_resource 创建
std::pmr::polymorphic_allocator<int> alloc(&pool);

// 分配
int* p = alloc.allocate(10);

// 构造
alloc.construct(p, 42);

// 销毁
alloc.destroy(p);

// 释放
alloc.deallocate(p, 10);
```

### 容器传播

```cpp
std::pmr::vector<std::pmr::vector<int>> nested(&pool);

// 内层 vector 继承 allocator
nested.emplace_back();  // 内层也使用 pool
```

---

## 自定义 memory_resource

### 实现示例

```cpp
class FixedPool : public std::pmr::memory_resource {
    void* do_allocate(size_t bytes, size_t alignment) override {
        // 自定义分配逻辑
        return ptr_;
    }
    
    void do_deallocate(void* p, size_t bytes, size_t alignment) override {
        // 自定义释放逻辑（或忽略）
    }
    
    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }
};
```

---

## Prism PMR 使用

### 热路径

协程和协议处理使用 PMR：

| 场景 | 资源类型 | 详见 |
|------|----------|------|
| 协程帧临时 | monotonic_buffer | [[dev/coding/pmr|PMR 使用规范]] |
| 连接池 | synchronized_pool | [[core/channel/overview|Channel]] |
| 协议处理 | monotonic_buffer | [[core/pipeline/overview|Pipeline]] |

### 规范要求

1. 热路径必须使用 PMR
2. 临时数据优先 monotonic_buffer
3. 长生命周期数据用 pool_resource

详见 [[dev/coding/pmr|PMR 使用规范]]。

---

## 性能对比

| Allocator | 分配时间 | 内存碎片 | 线程安全 |
|-----------|----------|----------|----------|
| new/delete | 慢 | 高 | 安全 |
| monotonic_buffer | 极快 | 无 | 需外部保证 |
| unsynchronized_pool | 快 | 低 | 不安全 |
| synchronized_pool | 较快 | 低 | 安全 |

---

## 相关参考

- [[core/memory/overview|Memory 模块]] — Prism 内存管理实现
- [[dev/coding/pmr|PMR 使用规范]] — Prism PMR 规范
- [[cpp23-coroutine|C++23 协程]] — 协程内存管理

---

## 进一步阅读

- PMR: https://en.cppreference.com/w/cpp/memory/polymorphic_allocator
- memory_resource: https://en.cppreference.com/w/cpp/memory/memory_resource