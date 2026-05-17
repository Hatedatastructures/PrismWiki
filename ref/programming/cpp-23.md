---
title: C++23 新特性
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [cpp, cpp23, features]
---

# C++23 新特性

> C++23 引入了多项新特性，包括 std::expected、std::generator、consteval 等。

---

## 主要新特性

### std::expected<T, E>
错误处理类型，替代异常或错误码：

```cpp
std::expected<int, std::string> process() {
    if (error) return std::unexpected("error");
    return 42;
}
```

详见 [[ref/programming/error-handling|错误处理策略]]。

### std::generator<T>
协程生成器：

```cpp
std::generator<int> sequence() {
    for (int i = 0; i < 10; ++i)
        co_yield i;
}
```

### consteval
强制编译期计算：

```cpp
consteval int square(int x) { return x * x; }
int a = square(10);  // 必须编译期计算
```

详见 [[ref/programming/constexpr|Constexpr 计算]]。

---

## Prism 使用的 C++23 特性

- 协程（std::generator 模式）
- std::expected（错误处理）
- consteval（编译期常量）

---

## 相关参考

- C++23 参考: https://en.cppreference.com/w/cpp/23
- [[ref/programming/cpp23-coroutine|C++23 协程]]
- [[ref/programming/error-handling|错误处理策略]]