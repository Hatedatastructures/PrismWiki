---
tags: [memory, container]
layer: core
module: memory
source: include/prism/memory/container.hpp
title: memory::container
---

# memory::container

PMR 容器别名定义。所有容器使用 `polymorphic_allocator`，支持运行时切换内存资源。

## 核心类型

| 别名 | 标准库对应 |
|------|-----------|
| `memory::string` | `std::pmr::string` |
| `memory::vector<T>` | `std::pmr::vector<T>` |
| `memory::list<T>` | `std::pmr::list<T>` |
| `memory::map<K,V>` | `std::pmr::map<K,V>` |
| `memory::unordered_map<K,V>` | `std::pmr::unordered_map<K,V>` |
| `memory::unordered_set<K>` | `std::pmr::unordered_set<K>` |

## 辅助函数

| 函数 | 说明 |
|------|------|
| `current_resource()` | 返回 `std::pmr::get_default_resource()` |
| `effective_mr(mr)` | `mr ? mr : current_resource()`，替代三元模式 |

## 设计决策

### 为什么用别名而非 using namespace std::pmr？

`using namespace` 会导入所有 `std::pmr` 名称（包括 `memory_resource`、`polymorphic_allocator` 等），污染命名空间。别名方式只导入项目需要的类型，且 `memory::string` 比 `std::pmr::string` 更简洁。

**后果**: 新增容器类型需在此文件添加别名行。

## 使用场景

| 场景 | 推荐容器 | 内存来源 |
|------|---------|---------|
| 帧解析临时缓冲 | `memory::vector<byte>` | `frame_arena::get()` |
| 请求级字符串 | `memory::string` | `thread_local_pool()` |
| 会话索引 | `memory::unordered_map` | `global_pool()` |
| 路由表 | `memory::map` | `global_pool()` |

## 引用关系

### 被引用

- 全系统所有模块使用 `memory::string` / `memory::vector` 等容器
- [[core/memory/pool|pool]]：内存池使用这些容器
