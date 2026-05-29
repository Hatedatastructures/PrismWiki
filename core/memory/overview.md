---
tags: [memory, overview]
layer: core
module: memory
source: include/prism/memory/pool.hpp
  - include/prism/memory/container.hpp
  - include/prism/memory/pool.hpp
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

## 故障场景

### 1. enable_pooling() 未调用

**触发条件**: main.cpp 启动时跳过了 `system::enable_pooling()` 调用

**传播路径**: `get_default_resource()` 返回 `new_delete_resource` -> 所有 PMR 容器使用默认堆分配器 -> 热路径产生大量小对象 malloc/free

**外部表现**: 性能严重退化，CPU 时间花在全局堆锁竞争上

**恢复机制**: 无运行时恢复，必须重启进程

### 2. frame_arena 跨线程使用

**触发条件**: frame_arena 在线程 A 创建，在线程 B 调用 `get()` 或 `reset()`

**传播路径**: `monotonic_buffer_resource` 内部状态无锁保护 -> 数据竞争（UB）

**外部表现**: 内存损坏、段错误，或间歇性崩溃

### 3. 大对象穿透池

**触发条件**: 单次分配超过 `policy::max_size`（16384 字节 / 16KB）

**传播路径**: `pooled_object::operator new` 判断 size > max_size -> 回退到 `::operator new` -> 直接堆分配

**外部表现**: 不影响正确性，但该对象不享受池化收益（分配/释放走系统堆）

### 4. thread_local_pool 内存不释放

**触发条件**: worker 线程长期运行，池中累积大量 chunk

**传播路径**: `unsynchronized_pool_resource` 按需向 `new_delete_resource` 申请 chunk -> 不主动归还 -> 内存占用持续增长

**外部表现**: RSS 缓慢增长，但受 `max_blocks=256` 限制单线程池上限可控

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| memory -> std::pmr | 依赖 | PMR 基础设施（synchronized_pool, unsynchronized_pool, monotonic_buffer） |
| [[core/stats/overview|stats]] <- memory | 被依赖 | memory_tracker 包装全局 PMR 池，在分配/释放时更新计数器 |
| [[core/instance/overview|instance]] <- memory | 被依赖 | worker 持有 frame_arena，session 使用 hot_pool 分配临时对象 |
| [[core/transport/overview|transport]] <- memory | 被依赖 | 传输层使用 memory::vector 做缓冲区 |
| [[core/multiplex/overview|multiplex]] <- memory | 被依赖 | mux 帧解析使用 frame_arena |
| memory 自包含 | 无外部依赖 | 不依赖项目中其他模块，仅依赖标准库 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| policy::max_size 变更 | 所有池化对象 | 减小导致更多堆穿透，增大增加内存碎片 |
| policy::max_blocks 变更 | 池内存上限 | 减小降低峰值内存，增大允许更多 chunk |
| frame_arena 栈缓冲大小变更 | 帧解析路径 | 过小增加穿透频率，过大浪费栈空间 |
| enable_pooling() 行为变更 | 全局 PMR 默认资源 | 影响所有未显式指定分配器的 PMR 容器 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| C++ 标准 PMR 实现变更 | synchronized_pool / unsynchronized_pool 行为 | 线程安全保证和性能特征 |
| 编译器 thread_local 实现变更 | local_pool 的线程隔离语义 | 是否真正每线程独立 |

## 相关模块

- [[core/fault/overview|Fault 模块]]：错误码系统
- [[core/exception/overview|Exception 模块]]：异常系统
- [[core/stats/overview|Stats 模块]]：内存分配统计
- [[core/instance/overview|Instance 模块]]：worker 和 session 的内存使用
