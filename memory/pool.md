---
title: 内存池详解
module: memory::pool
namespace: psm::memory
files:
  - include/prism/memory/pool.hpp
see_also:
  - "[[memory]]"
  - "[[channel/transport]]"
  - "[[resolve/dns]]"
---

# 内存池详解

内存池模块是 Prism 热路径零堆分配的核心基础设施。采用三级内存池架构，
通过 PMR（多态内存资源）接口为所有热路径容器提供高效的内存分配。

## 三级内存池架构

```
  frame_arena (帧级, 栈上缓冲区 + 单调增长)
      │  上游溢出
      ▼
  thread_local_pool (线程级, 无锁 unsynchronized_pool)
      │  跨线程对象
      ▼
  global_pool (进程级, 线程安全 synchronized_pool)
```

### 1. global_pool — 全局同步池

```cpp
static synchronized_pool* system::global_pool()
```

- 使用 `new` 分配，随进程销毁（避免静态析构顺序问题）
- 类型为 `std::pmr::synchronized_pool_resource`
- 适用于跨线程传递的对象和生命周期不确定的长期对象
- 通过 `enable_global_pooling()` 设置为 `std::pmr::default_resource`

### 2. thread_local_pool — 线程局部无锁池

```cpp
static unsynchronized_pool* system::thread_local_pool()
```

- `thread_local` 存储，每个线程独立实例
- 类型为 `std::pmr::unsynchronized_pool_resource`
- 无锁设计，消除多线程竞争
- 适用于临时计算和单线程处理逻辑

### 3. hot_path_pool — 热路径池

```cpp
static unsynchronized_pool* system::hot_path_pool()
```

- `thread_local_pool()` 的语义化别名
- 分配的对象生命周期必须与当前线程绑定，禁止跨线程传递

### 4. frame_arena — 帧分配器

```cpp
class frame_arena {
    std::byte buffer_[512];           // 栈上缓冲区
    monotonic_buffer resource_;       // 单调增长资源
};
```

- 512 字节栈缓冲区覆盖典型 mux 地址头，避免穿透到上游池
- 上游为 `thread_local_pool()`，溢出时从线程池补充
- `get()` 返回 `resource_pointer` 供 PMR 容器使用
- `reset()` 释放所有已分配内存，重置游标
- 禁止拷贝和移动

## policy — 调参配置

```cpp
struct policy {
    static constexpr std::size_t max_blocks = 256;       // 每 Chunk 最大块数
    static constexpr std::size_t max_pool_size = 16384;  // 最大池化阈值（16KB）
};
```

- `max_blocks`：降低此值可减少内存峰值
- `max_pool_size`：16KB 足以覆盖 HTTP Header 等典型对象
- 超过 `max_pool_size` 的分配直通系统堆

两个参数同时应用于 `global_pool` 和 `thread_local_pool`。

## system — 全局内存系统管理器

`system` 类提供全局单例的内存池访问接口，所有方法均为静态方法。

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `global_pool()` | `synchronized_pool*` | 全局线程安全池 |
| `thread_local_pool()` | `unsynchronized_pool*` | 线程局部无锁池 |
| `hot_path_pool()` | `unsynchronized_pool*` | 热路径池（别名） |
| `enable_global_pooling()` | `void` | 将默认资源设为全局池 |

### 启动初始化

在 `main.cpp` 中必须尽早调用：

```cpp
memory::system::enable_global_pooling();
```

调用后，所有未指定显式内存资源的 PMR 容器自动使用 `global_pool()`。

## pooled_object — 对象池基类

CRTP 模板类，通过重载 `operator new` / `operator delete` 使继承类
自动使用内存池分配。

```cpp
template <typename T, pool_type Type = pool_type::local>
class pooled_object;
```

### pool_type 枚举

| 值 | 池 | 适用场景 |
|----|------|----------|
| `global` | `global_pool()` | 跨线程传递对象 |
| `local` | `thread_local_pool()` | 单线程热路径对象（默认） |

### 分配策略与使用

小对象（≤ `policy::max_pool_size`，即 16KB）从目标池分配，大对象直通系统堆。
`operator delete` 必须与 `operator new` 对应。同时提供 `new[]`/`delete[]` 数组版本。

使用：继承 `pooled_object<T, pool_type::local>` 即可，`operator new/delete` 自动生效。

## frame_arena 使用模式

`frame_arena` 适用于短生命周期、高频分配的场景，典型用法：

```cpp
// 协程入口处构造
frame_arena arena;

// 使用 arena 的内存资源创建 PMR 容器
memory::vector<std::byte> header(arena.get());
memory::string host(arena.get());

// ... 处理逻辑 ...

// 处理完毕后重置（或依赖析构函数）
arena.reset();
```

**分配顺序：** 先从 512 字节栈缓冲区分配（零系统调用），
耗尽后从 `thread_local_pool()` 补充。`reset()` 仅重置游标。

**约束：** 分配的内存必须在 `reset()` 或析构前使用，
禁止跨协程/线程传递，禁止拷贝和移动。

## 典型使用场景

| 场景 | 推荐池 | 原因 |
|------|--------|------|
| session 对象 | `pooled_object<session>` | 热路径频繁创建销毁 |
| DNS 缓存条目 | `global_pool` | 跨线程共享 |
| 协程临时缓冲区 | `frame_arena` | 短生命周期，高频分配 |
| 连接池管理 | `global_pool` | 跨 worker 线程共享 |
| 协议解析临时数据 | `thread_local_pool` | 单线程处理 |

## 与其他模块的关系

- [[memory]]：`memory::string`、`memory::vector` 等类型默认使用 `global_pool`（调用 `enable_global_pooling()` 后）
- [[resolve/dns]]：DNS 缓存和规则引擎的容器通过 PMR 分配器使用内存池
- [[channel/transport]]：`snapshot` 的捕获缓冲区使用 `memory::vector<std::byte>`
- [[memory/pool]] 启动时必须先于所有使用 PMR 容器的模块初始化
