---
title: Constexpr Computation
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [programming, cpp, constexpr, compile-time]
---

# Constexpr 计算

> C++ 编译期计算原理、constexpr 函数、常量表达式。
> 最后更新：2026-05-17

---

## constexpr 概念

### 定义

`constexpr` 指示编译期可计算的值或函数：

```cpp
// 编译期常量
constexpr int max_size = 1024;
constexpr double pi = 3.14159265358979;

// 编译期函数
constexpr int square(int x) {
    return x * x;
}
constexpr int result = square(10);  // 编译期计算
```

### 编译期 vs 运行期

| 场景 | 执行时机 | 优势 |
|------|----------|------|
| 编译期 | 构建时 | 零运行开销、类型安全 |
| 运行期 | 运行时 | 动态灵活性 |

---

## constexpr 变量

### 规则

```cpp
constexpr int a = 42;           // OK：字面量
constexpr int b = a + 10;       // OK：constexpr 表达式
constexpr int c = rand();       // 错误：运行期函数

// 浮点数
constexpr double d = 1.0 / 3.0; // OK：编译期计算
```

### 与 const 的区别

```cpp
const int a = rand();           // OK：运行期初始化
constexpr int b = rand();       // 错误

const int c = 42;               // 运行期常量
constexpr int d = 42;           // 编译期常量

// 数组大小
int arr1[c];                    // 可能错误（非编译期）
int arr2[d];                    // OK：编译期大小
```

---

## constexpr 函数

### C++11 规则

- 单条 return 语句
- 只调用 constexpr 函数
- 只使用 constexpr 变量

### C++14+ 放宽

```cpp
constexpr int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

constexpr int sum(int n) {
    int result = 0;
    for (int i = 1; i <= n; ++i) {
        result += i;
    }
    return result;
}
```

### 允许的内容

| C++14 | C++20 | C++23 |
|-------|-------|-------|
| 局部变量 | `std::vector` | `std::optional` |
| 循环 | `std::string` | `std::unique_ptr` |
| 条件分支 | 动态分配 | `std::shared_ptr` |
| 多条语句 | `new/delete` | 更宽松 |

---

## constexpr 类

### 构造函数

```cpp
class Point {
    int x_, y_;
public:
    constexpr Point(int x, int y) : x_(x), y_(y) {}
    constexpr int x() const { return x_; }
    constexpr int y() const { return y_; }
    constexpr int distance() const { return x_ * x_ + y_ * y_; }
};

constexpr Point p(3, 4);
constexpr int d = p.distance();  // 编译期计算
```

### constexpr 方法

```cpp
constexpr Point operator+(Point a, Point b) {
    return Point(a.x() + b.x(), a.y() + b.y());
}

constexpr Point sum = Point(1, 2) + Point(3, 4);
```

---

## 编译期编程

### 模板元编程替代

```cpp
// 旧方式：模板
template<int N>
struct Factorial {
    static const int value = N * Factorial<N-1>::value;
};
template<>
struct Factorial<0> { static const int value = 1; };

// 新方式：constexpr
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

// 使用
int arr[Factorial<5>::value];    // 模板
int arr[factorial(5)];           // constexpr（更简洁）
```

### 类型计算

```cpp
constexpr size_t type_size = sizeof(int);
constexpr size_t alignment = alignof(double);

// 模板参数推导
template<typename T>
constexpr bool is_small = sizeof(T) <= sizeof(void*);
```

---

## Prism 应用

### 协议常量

```cpp
// 编译期协议常量
constexpr size_t SOCKS5_HEADER_SIZE = 3;
constexpr size_t TROJAN_PASSWORD_HASH_SIZE = 56;
constexpr size_t VLESS_UUID_SIZE = 36;

// 帧格式
constexpr uint8_t SOCKS5_VERSION = 0x05;
constexpr uint8_t SOCKS5_CONNECT = 0x01;
constexpr uint8_t SOCKS5_NO_AUTH = 0x00;
```

### 编译期校验

```cpp
// 配置校验
constexpr bool valid_port(int port) {
    return port >= 1 && port <= 65535;
}

// 使用
static_assert(valid_port(8080), "invalid port");
```

### 加密常量

```cpp
// AEAD 密钥长度
constexpr size_t AES_128_GCM_KEY_SIZE = 16;
constexpr size_t AES_256_GCM_KEY_SIZE = 32;
constexpr size_t CHACHA20_POLY1305_KEY_SIZE = 32;

// nonce 长度
constexpr size_t AES_GCM_NONCE_SIZE = 12;
constexpr size_t CHACHA20_NONCE_SIZE = 12;
```

---

## consteval（C++23）

### 强制编译期

```cpp
// consteval：必须在编译期计算
consteval int square(int x) {
    return x * x;
}

constexpr int a = square(10);   // OK
int b = square(10);             // OK：编译期计算
int c = square(rand());         // 错误：无法编译期计算
```

### 与 constexpr 区别

```cpp
constexpr int f(int x) { return x * 2; }
consteval int g(int x) { return x * 2; }

int a = f(rand());              // OK：运行期计算
int b = g(rand());              // 错误：必须编译期
```

---

## if constexpr

### 编译期条件

```cpp
template<typename T>
void process(T value) {
    if constexpr (std::is_integral_v<T>) {
        // 整数处理
    } else if constexpr (std::is_floating_point_v<T>) {
        // 浮点处理
    } else {
        // 其他类型
    }
}
```

### SFINAE 替代

```cpp
// 旧方式：SFINAE
template<typename T, std::enable_if_t<std::is_integral_v<T>, int> = 0>
void process(T value) { /*...*/ }

// 新方式：if constexpr
template<typename T>
void process(T value) {
    if constexpr (std::is_integral_v<T>) { /*...*/ }
}
```

---

## 相关参考

- [[cpp23-coroutine|C++23 协程]] — 协程特性
- [[pmr-concepts|PMR 概念]] — 内存管理
- [[core/protocol/overview|Protocol 模块]] — 协议常量

---

## 进一步阅读

- Constexpr: https://en.cppreference.com/w/cpp/language/constexpr
- Consteval: https://en.cppreference.com/w/cpp/language/consteval
- C++23 特性: https://en.cppreference.com/w/cpp/23