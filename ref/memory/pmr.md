---
title: "多态内存资源 (PMR)"
category: "memory"
type: ref
tags: [内存, pmr, 分配器]
created: 2026-05-15
updated: 2026-05-15
---

# 多态内存资源 (PMR)

**类别**: 内存

## 概述

PMR（Polymorphic Memory Resource）是 C++17 引入的内存分配抽象，允许容器使用不同的内存分配策略。Prism 使用 PMR 实现热路径零堆分配。

## 原理

### 核心概念

- **memory_resource**: 内存资源基类
- **polymorphic_allocator**: 多态分配器
- **pmr::vector/string**: 使用 PMR 分配器的容器

### 内存池

Prism 实现了全局内存池：
- **pool**: 固定大小块的内存池
- **arena**: 线性分配的竞技场

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| container | PMR 容器定义 | [[memory/container|container]] |
| pool | 内存池实现 | [[memory/pool|pool]] |

## 参考资料

- [C++17 PMR](https://en.cppreference.com/w/cpp/memory)

## 相关知识
- [[ref/memory/arena|竞技场分配器]] — Arena 分配器
- [[ref/memory/zero-copy|零拷贝]] — 零拷贝技术
