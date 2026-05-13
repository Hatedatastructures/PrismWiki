---
title: Memory 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [memory, pmr, architecture, infrastructure, runtime]
related: [[agent/overview], [architecture], [channel/overview], [crypto]]
module: memory
category: infrastructure
layer: runtime
status: stable
dependencies:
  - std::pmr
dependents:
  - channel
  - multiplex
  - stealth
  - crypto
  - protocol
  - agent
---

# Memory 模块

## 1. 模块定位

Memory 模块是 Prism 的**内存管理基础设施**，基于 C++ PMR（Polymorphic Memory Resources）实现热路径零堆分配策略。它为整个系统提供统一的内存分配接口，消除热路径中的 `malloc/free` 竞争开销。

**核心价值**：让所有热路径容器自动走池分配，开发者无需手动管理内存资源。

**边界**：Memory 模块是 header-only 的纯基础设施，不包含业务逻辑。

## 2. 核心功能

### 2.1 PMR 容器别名

提供统一的容器类型别名，所有热路径数据结构均使用 PMR 分配器：

| 别名 | 底层类型 | 用途 |
|------|---------|------|
| `memory::string` | `std::pmr::string` | 字符串 |
| `memory::vector<T>` | `std::pmr::vector<T>` | 动态数组 |
| `memory::list<T>` | `std::pmr::list<T>` | 双向链表 |
| `memory::map<K,V>` | `std::pmr::map<K,V>` | 有序映射 |
| `memory::unordered_map<K,V>` | `std::pmr::unordered_map<K,V>` | 哈希映射 |
| `memory::unordered_set<K>` | `std::pmr::unordered_set<K>` | 哈希集合 |

### 2.2 三级内存池

| 层级 | 池类型 | 线程安全 | 访问方式 |
|------|--------|---------|---------|
| L1 | `synchronized_pool_resource` | ✅ | `system::global_pool()` |
| L2 | `unsynchronized_pool_resource` | ❌ | `system::thread_local_pool()` |
| L3 | `monotonic_buffer_resource` | ❌ | `frame_arena` |

### 2.3 对象池基类

`pooled_object<T>` 通过 CRTP 重载 `operator new/delete`，子类自动使用内存池分配。小对象（≤16KB）走池，大对象直通系统堆。

### 2.4 帧分配器

`frame_arena` 使用 512 字节栈缓冲区 + 单调增长资源，适合短生命周期、高频分配场景（如 mux 地址头解析）。

## 3. 关键类和接口

### 3.1 PMR 类型别名（`container.hpp`）

```cpp
namespace psm::memory {
    using resource = std::pmr::memory_resource;
    using resource_pointer = std::add_pointer_t<resource>;
    using synchronized_pool = std::pmr::synchronized_pool_resource;
    using unsynchronized_pool = std::pmr::unsynchronized_pool_resource;
    using monotonic_buffer = std::pmr::monotonic_buffer_resource;

    template<typename T> using allocator = std::pmr::polymorphic_allocator<T>;
    using string = std::pmr::string;
    template<typename V> using vector = std::pmr::vector<V>;
    template<typename V> using list = std::pmr::list<V>;
    template<typename K, typename V, typename C = std::less<K>>
        using map = std::pmr::map<K, V, C>;
    template<typename K, typename V, typename H = std::hash<K>, typename E = std::equal_to<K>>
        using unordered_map = std::pmr::unordered_map<K, V, H, E>;
    template<typename K, typename H = std::hash<K>, typename E = std::equal_to<K>>
        using unordered_set = std::pmr::unordered_set<K, H, E>;

    inline auto current_resource() -> resource_pointer;
}
```

`current_resource()` 是 `std::pmr::get_default_resource()` 的包装，调用 `enable_global_pooling()` 后返回全局池。

### 3.2 `system` — 全局内存系统管理器（`pool.hpp`）

```cpp
class system {
public:
    static synchronized_pool *global_pool();       // 全局线程安全池（new 出来，随进程销毁）
    static unsynchronized_pool *thread_local_pool(); // 线程局部无锁池
    static unsynchronized_pool *hot_path_pool();    // = thread_local_pool()，语义别名
    static void enable_global_pooling();            // set_default_resource(global_pool())
};
```

全局池使用 `new` 分配（非静态变量），避免静态析构顺序问题。pool_options 为 `max_blocks=256, max_pool_size=16384`。

### 3.3 `policy` — 内存策略配置（`pool.hpp`）

```cpp
struct policy {
    static constexpr std::size_t max_blocks = 256;
    static constexpr std::size_t max_pool_size = 16384;  // 16KB 池化阈值
};
```

### 3.4 `pooled_object<T, Type>` — 对象池基类（`pool.hpp`）

```cpp
enum class pool_type { global, local };  // global=跨线程, local=单线程热路径

template<typename T, pool_type Type = pool_type::local>
class pooled_object {
public:
    static resource_pointer get_target_pool();
    void *operator new(std::size_t count);       // ≤16KB 走池，>16KB 走堆
    void operator delete(void *ptr, std::size_t count);
};
```

默认使用 `pool_type::local`（无锁），跨线程传递对象需显式指定 `pool_type::global`。

### 3.5 `frame_arena` — 帧分配器（`pool.hpp`）

```cpp
class frame_arena {
    std::byte buffer_[512];           // 栈上缓冲区
    monotonic_buffer resource_;       // 单调增长资源
public:
    frame_arena();                    // 使用栈缓冲区 + thread_local_pool 作为上游
    resource_pointer get();           // 获取内存资源指针
    void reset();                     // 释放所有已分配内存
};
```

512 字节栈缓冲区覆盖典型 mux 地址头，超出时穿透到线程局部池。不可拷贝移动，`reset()` 一次性释放所有内存。

## 4. 三级内存池架构

```
┌─────────────────────────────────────────────────┐
│  Level 1: 全局同步池 (global_pool)                │
│  - synchronized_pool_resource, 线程安全           │
│  - 跨线程传递的对象、长期对象                      │
├─────────────────────────────────────────────────┤
│  Level 2: 线程局部无锁池 (thread_local_pool)      │
│  - unsynchronized_pool_resource, 无锁             │
│  - 热路径临时对象、单线程处理                       │
├─────────────────────────────────────────────────┤
│  Level 3: 帧分配器 (frame_arena)                  │
│  - 512B 栈缓冲区 + monotonic_buffer              │
│  - mux 地址头等短生命周期对象                       │
└─────────────────────────────────────────────────┘
```

## 5. 内存分配决策流程

```
对象分配请求
    ├── PMR 容器（memory::vector 等）
    │   ├── 有显式资源 → 使用指定资源
    │   └── 无显式资源 → default_resource
    │       ├── enable_global_pooling() 已调用 → global_pool()
    │       └── 未调用 → new_delete_resource()（系统堆）
    └── pooled_object 子类
        ├── size ≤ 16KB → 目标池分配
        └── size > 16KB → 系统堆
```

## 6. 启动初始化

```cpp
int main() {
    memory::system::enable_global_pooling();  // 必须最先调用
    // 之后所有未指定显式资源的 PMR 容器自动使用 global_pool()
}
```

## 7. 文件清单

### 头文件（`include/prism/memory/`）

```
memory.hpp                           # 聚合头文件
├── container.hpp                    # PMR 容器别名、资源类型、current_resource()
└── pool.hpp                         # system, policy, pool_type, pooled_object, frame_arena
```

**注意**：Memory 模块是 header-only，无源文件（`src/prism/memory/` 不存在）。

## 8. 设计原理

- **PMR 多态分配器**：运行时切换内存资源，无需为每种分配器实例化模板
- **线程封闭优化**：热路径容器默认使用 `thread_local_pool`（无锁），跨线程才用 `global_pool`
- **new 出来的全局池**：避免静态析构顺序问题，随进程销毁
- **16KB 池化阈值**：覆盖 HTTP Header 等典型对象，大对象直通系统堆

## 相关页面

- [[agent]] — Agent 模块（启动时调用 enable_global_pooling）
- [[channel]] — Channel 模块（连接池使用 PMR 容器）
- [[crypto]] — Crypto 模块（加密缓冲区使用帧分配器）
- [[architecture]] — 架构设计
