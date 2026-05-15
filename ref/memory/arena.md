---
title: "竞技场分配器 (Arena)"
category: "memory"
type: ref
tags: [内存, arena, 分配器]
created: 2026-05-15
updated: 2026-05-15
---

# 竞技场分配器 (Arena)

**类别**: 内存

## 概述

Arena 分配器是一种线性分配的内存分配器，只支持分配，不支持单个释放。适合生命周期一致的临时对象。

## 原理

### 工作流程

```
1. 预分配一大块内存
2. 分配时移动指针
3. 所有对象一起释放
```

### 优势

- 分配速度极快（移动指针）
- 无内存碎片
- 缓存友好

## 在 Prism 中的应用

Prism 使用 Arena 分配器管理协程帧中的临时对象。

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| pool | 内存池 | [[memory/pool|pool]] |

## 参考资料

- [Arena Allocation](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-003/)

## 相关知识
- [[ref/memory/pmr|PMR]] — 多态内存资源
