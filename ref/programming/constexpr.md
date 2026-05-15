---
title: "constexpr 编译期计算"
category: "programming"
type: ref
tags: [编程, c++, constexpr, 编译期]
created: 2026-05-15
updated: 2026-05-15
---

# constexpr 编译期计算

**类别**: 编程

## 概述

constexpr 是 C++11 引入的关键字，允许在编译期计算表达式的值。Prism 使用 constexpr 实现编译期常量和优化。

## 原理

### 使用场景

- **编译期常量**: `constexpr int N = 42;`
- **编译期函数**: `constexpr int factorial(int n) { ... }`
- **编译期类**: `constexpr struct { ... }`

### C++23 扩展

- **constexpr 更多标准库**: 更多函数可在编译期调用
- **if consteval**: 编译期分支

## 在 Prism 中的应用

Prism 使用 constexpr 实现编译期常量和配置。

## 参考资料

- [constexpr](https://en.cppreference.com/w/cpp/language/constexpr)

## 相关知识
- [[ref/programming/c++23-coroutines|C++23 协程]] — C++23 特性
