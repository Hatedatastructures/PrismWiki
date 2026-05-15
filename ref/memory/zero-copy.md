---
title: "零拷贝"
category: "memory"
type: ref
tags: [内存, 零拷贝, 性能]
created: 2026-05-15
updated: 2026-05-15
---

# 零拷贝

**类别**: 内存

## 概述

零拷贝是一种避免数据在内存中不必要复制的技术，提高数据传输效率。

## 原理

### 传统拷贝

```
磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡
```

### 零拷贝

```
磁盘 → 内核缓冲区 → 网卡
```

### 技术

- **sendfile**: 直接在内核空间传输
- **mmap**: 内存映射
- **splice**: 管道传输

## 在 Prism 中的应用

Prism 使用零拷贝技术优化数据传输。

| 概念 | 说明 | 链接 |
|------|------|------|
| PMR 容器 | 使用多态内存资源减少拷贝 | [[memory/container|container]] |
| 传输层 | 异步读写避免不必要的数据拷贝 | [[channel/transport/transmission|transmission]] |

## 参考资料

- [Zero-copy](https://en.wikipedia.org/wiki/Zero-copy)

## 相关知识
- [[ref/memory/pmr|PMR]] — 多态内存资源
