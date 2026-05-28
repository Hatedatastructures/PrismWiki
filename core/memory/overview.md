---
tags: [memory, overview]
layer: core
module: memory
source:
  - I:/code/Prism/include/prism/memory/container.hpp
  - I:/code/Prism/include/prism/memory/pool.hpp
title: Memory 模块
---

# Memory 模块

基于 C++17 PMR (Polymorphic Memory Resource) 的内存管理基础设施。设计目标：热路径零分配、线程封闭、大小分类。

## 设计决策

### 为什么用 PMR 而非自定义分配器模板？

标准库的 `std::allocator<T>` 是编译期绑定，切换内存源需改动模板参数。PMR 的 `polymorphic_allocator<T>` 在运行时绑定 `memory_resource` 指针，容器类型不随分配器变化（`memory::vector<int>` 不需要模板参数指定分配器）。同一类型可使用不同内存源，适合请求级别切换分配策略。

**后果**: PMR 有一次虚函数调用开销（`memory_resource::allocate()` 是虚函数），但比 `new/delete` 的全局锁竞争快得多。

### 为什么 thread_local_pool 用 new 而非静态局部变量？

`unsynchronized_pool` 实例通过 `new` 分配（`pool.hpp:81`），因为 `thread_local` 的析构时机不确定。如果使用静态局部变量，在线程退出时析构顺序可能与池中对象冲突。`new` 出来的实例随进程销毁，避免析构顺序问题。

**后果**: `thread_local_pool` 在进程退出前不会释放，但这是可接受的（总量受限于线程数 × 池大小）。

### 为什么 frame_arena 栈缓冲区 512 字节？

典型 mux 帧头 + 地址信息 < 128 字节。512 字节覆盖绝大部分帧解析场景，避免穿透到上游池。超过 512 字节的分配自动 fallback 到 `thread_local_pool`，不丢数据。

**后果**: 如果某次解析需要 > 512 字节的临时空间，会穿透一次池分配，但 `reset()` 后 monotonic_buffer 保留已分配的 chunk 可复用。

## 模块组成

| 组件 | 说明 | 源码 |
|------|------|------|
| [[core/memory/container\|container]] | PMR 容器别名 | `prism/memory/container.hpp` |
| [[core/memory/pool\|pool]] | 内存池系统 | `prism/memory/pool.hpp` |

## 核心组件

| 组件 | 说明 | 延迟 |
|------|------|------|
| `system::global_pool()` | 线程安全池（有锁） | ~15-25ns |
| `system::local_pool()` | 线程局部池（无锁） | ~3-5ns |
| `system::hot_pool()` | `local_pool()` 的别名 | ~3-5ns |
| `pooled_object<T>` | CRTP 基类，自动 new/delete 池化 | 同池 |
| `frame_arena` | 栈上 512B + monotonic_buffer | < 2ns |

## 约束

### enable_pooling() 必须最先调用

**类型**: 调用顺序

**规则**: `system::enable_pooling()` 必须在创建任何 PMR 容器之前调用（`main()` 开头）。

**违反后果**: 在调用前创建的容器使用默认堆分配器，之后的容器使用全局池。不同分配器的容器交换/移动时可能导致悬挂指针。

**源码依据**: `pool.hpp:104-107`

### frame_arena 禁止跨线程

**类型**: 线程安全

**规则**: `frame_arena` 使用 `monotonic_buffer`（非线程安全）且引用 `thread_local_pool` 作为上游资源，必须在同一线程内创建、使用、reset。

**违反后果**: 数据竞争。

**源码依据**: `pool.hpp:231-232`

## 相关模块

- [[core/fault/overview|Fault 模块]]：错误码系统
- [[core/exception/overview|Exception 模块]]：异常系统
