---
tags: [memory, pool]
layer: core
module: memory
source: include/prism/memory/pool.hpp
title: memory::pool
---

# memory::pool

内存池系统，提供全局/线程局部池管理、CRTP 池化基类、帧分配器。

## 核心组件

### system

| 方法 | 返回 | 说明 |
|------|------|------|
| `global_pool()` | `synchronized_pool*` | 线程安全池，跨线程对象 |
| `local_pool()` | `unsynchronized_pool*` | 线程局部无锁池 |
| `hot_pool()` | `unsynchronized_pool*` | `local_pool()` 的语义别名 |
| `enable_pooling()` | void | 设置默认资源为 `global_pool()` |

### policy

| 常量 | 值 | 说明 |
|------|----|------|
| `max_blocks` | 256 | 每 chunk 最大块数 |
| `max_size` | 16384 | 池化阈值（16KB），超过直通系统堆 |

### pooled_object<T, pool_type>

CRTP 基类，重载 `operator new/delete`。默认 `pool_type::local`（无锁），跨线程对象用 `pool_type::global`。

### frame_arena

栈上 512B 缓冲区 + `monotonic_buffer`。`get()` 返回 `resource_pointer`，`reset()` 一次性释放。

## 设计决策

### 为什么 global_pool 用 new 而非静态局部变量？

`global_pool()` 通过 `new synchronized_pool(...)` 创建（`pool.hpp:60`）。静态局部变量在程序退出时析构，如果其他静态对象（如日志系统）在析构阶段仍在使用池，会访问已析构的对象。`new` 出来的池随进程销毁，不存在析构顺序问题。

**后果**: `global_pool()` 永不释放，但全局只有一个实例（约 2MB），开销可忽略。

### 为什么 hot_pool 是 local_pool 的别名而非独立池？

两个概念指向同一个物理池：`hot_pool()` 强调"仅限热路径使用"的语义约束，`local_pool()` 强调"线程局部"的实现特征。如果创建独立池，两个池之间无法共享空闲内存块，增加内存占用。

**后果**: `hot_pool()` 返回的指针与 `local_pool()` 完全相同，可以互相替代。

### 为什么 frame_arena 的 upstream 是 local_pool 而非 new_delete_resource？

当 512B 栈缓冲区用尽时，monotonic_buffer 向上游请求新 chunk。如果上游是 `new_delete_resource`，每次请求都是系统堆分配。使用 `local_pool`（无锁池），chunk 从池中快速分配，且 `reset()` 后 monotonic_buffer 保留的 chunk 可复用。

**后果**: frame_arena 依赖线程局部池的存在，不能在池初始化前使用。

## 约束

### enable_pooling() 调用时序

**类型**: 调用顺序

**规则**: `system::enable_pooling()` 必须在 `main()` 最开始调用，早于任何 PMR 容器创建。

**违反后果**: 在调用前创建的容器使用默认堆分配器，调用后创建的使用全局池。不同分配器的容器交互可能导致问题。

**源码依据**: `pool.hpp:104-107`

### frame_arena 生命周期

**类型**: 生命周期

**规则**: `frame_arena::get()` 返回的 `resource_pointer` 仅在 `frame_arena` 存活期间有效。`reset()` 后之前分配的所有内存失效。

**违反后果**: 悬挂指针。

**源码依据**: `pool.hpp:240-247`

### pooled_object 默认 local 不跨线程

**类型**: 线程安全

**规则**: `pooled_object<T>` 默认使用 `pool_type::local`，从 `thread_local_pool` 分配。跨线程传递需显式指定 `pool_type::global`。

**违反后果**: 对象从 A 线程的池分配，在 B 线程释放，导致池状态损坏。

**源码依据**: `pool.hpp:132`

## 使用场景

| 场景 | 组件 | 池类型 |
|------|------|--------|
| Session 对象 | `pooled_object<Session>` | local |
| 全局配置 | `pooled_object<Config, pool_type::global>` | global |
| 帧解析 | `frame_arena` | 栈 + local |
| 热路径临时容器 | `memory::vector<byte>{local_pool()}` | local |

## 引用关系

### 依赖

- [[core/memory/container|container]]：PMR 容器别名

### 被引用

- 几乎所有模块使用 `system::local_pool()` / `global_pool()`
- [[core/memory/overview|Memory 模块]]：总览
