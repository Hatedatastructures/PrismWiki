---
title: C++23 Coroutine
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [programming, cpp, coroutine, cpp23]
---

# C++23 协程

> C++20/23 协程原理、编译器支持、使用模式。
> 最后更新：2026-05-17

---

## 协程概念

### 定义

协程是可以暂停执行并在稍后恢复的函数：

```
普通函数:
start → ... → end（一次性执行完毕）

协程:
start → ... → suspend → ... → resume → ... → suspend → ... → end
```

### 与线程的区别

| 特性 | 线程 | 协程 |
|------|------|------|
| 调度 | 操作系统抢占式 | 用户合作式 |
| 上下文切换 | 内核态（昂贵） | 用户态（轻量） |
| 内存占用 | MB 级栈空间 | KB 帧空间 |
| 并发数 | 受限于线程数 | 可达百万级 |

---

## 协程机制

### 关键字

| 关键字 | 作用 |
|--------|------|
| `co_await` | 暂停协程，等待异步操作 |
| `co_yield` | 暂停协程，返回值 |
| `co_return` | 结束协程，返回最终值 |

### Promise Type

每个协程关联一个 promise 对象：

```cpp
// Promise 接口
template<typename T>
struct promise_type {
    T get_return_object();        // 创建返回对象
    std::suspend_always initial_suspend();  // 初始暂停
    std::suspend_always final_suspend();    // 最终暂停
    void return_value(T value);   // co_return 处理
    void unhandled_exception();   // 异常处理
    
    // co_await 转换
    auto await_transform(Awaitable a);
};
```

### Awaitable

`co_await` 操作的对象：

```cpp
// Awaitable 接口
struct Awaitable {
    bool await_ready();           // 是否立即完成
    void await_suspend(std::coroutine_handle<> h); // 暂停时调用
    T await_resume();             // 恢复时返回值
};
```

---

## 协程流程

### 执行过程

```
1. 调用协程函数
   │
2. 分配协程帧（堆）
   │
3. 构造 promise 对象
   │
4. 调用 promise.get_return_object()
   │
5. 调用 promise.initial_suspend()
   │
   ├─ suspend_never: 立即开始执行
   └─ suspend_always: 暂停，等待首次 resume
   │
6. 执行协程体
   │
7. 遇到 co_await
   │
   ├─ await_ready() == true: 不暂停，继续执行
   └─ await_ready() == false:
       │
       调用 await_suspend(handle)
       │
       暂停协程，保存状态
       │
       ... 外部操作完成 ...
       │
       调用 handle.resume()
       │
       调用 await_resume()，获取结果
       │
       继续执行协程体
   │
8. 遇到 co_return
   │
9. 调用 promise.return_value()
   │
10. 调用 promise.final_suspend()
    │
    ├─ suspend_never: 销毁协程帧
    └─ suspend_always: 暂停，等待显式销毁
```

### 协程帧

协程帧包含：

| 成员 | 说明 |
|------|------|
| promise 对象 | 协程状态管理 |
| 参数副本 | 协程参数存储 |
| 局部变量 | 暂停点的活跃变量 |
| 恢复点 | 下次执行的代码位置 |

---

## coroutine_handle

### 类型

```cpp
// 泛型 handle
std::coroutine_handle<> handle;

// 带 promise 的 handle
std::coroutine_handle<MyPromise> handle;
```

### 操作

```cpp
// 从 promise 获取
auto handle = std::coroutine_handle<MyPromise>::from_promise(promise);

// 恢复执行
handle.resume();

// 销毁帧
handle.destroy();

// 检查状态
bool done = handle.done();  // 是否到达 final_suspend
```

---

## 标准库支持

### C++20

- `<coroutine>` 头文件
- `std::coroutine_handle`
- `std::suspend_always` / `std::suspend_never`
- `std::noop_coroutine`

### C++23 新增

- `std::generator<T>` — 惰性序列生成器
- Deducing this — 显式对象参数

```cpp
// generator 示例（C++23）
std::generator<int> fibonacci(int n) {
    int a = 0, b = 1;
    for (int i = 0; i < n; ++i) {
        co_yield a;
        a = std::exchange(b, a + b);
    }
}

// 使用
for (int x : fibonacci(10)) {
    std::cout << x << "\n";
}
```

---

## 常见模式

### Task<T>

异步任务返回类型：

```cpp
template<typename T>
class Task {
    struct promise_type {
        T value;
        std::exception_ptr exception;
        
        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(T v) { value = v; }
        void unhandled_exception() { exception = std::current_exception(); }
    };
    
    std::coroutine_handle<promise_type> handle_;
    
    // awaitable
    bool await_ready() { return handle_.done(); }
    void await_suspend(std::coroutine_handle<> h) { /*...*/ }
    T await_resume() {
        if (handle_.promise().exception)
            std::rethrow_exception(handle_.promise().exception);
        return handle_.promise().value;
    }
};
```

### 异步生成器

```cpp
template<typename T>
class AsyncGenerator {
    // 惰性异步序列
    // co_yield 暂停，co_await 消费
};
```

---

## Prism 协程实践

### 纯度要求

Prism 协程必须是"纯协程"：

1. 所有异步操作必须用 `co_await`
2. 不可在协程内使用阻塞操作
3. 错误处理必须完整

详见 [[dev/coding/coroutine|协程约定]]。

### 示例

```cpp
// Prism 协程示例
asio::awaitable<void> handle_session(tcp::socket socket) {
    try {
        // 读取请求
        auto request = co_await read_request(socket);
        
        // 处理
        auto response = co_await process_request(request);
        
        // 写入响应
        co_await write_response(socket, response);
        
    } catch (const std::exception& e) {
        log::error("session error: {}", e.what());
    }
}
```

---

## 编译器支持

| 编译器 | 版本要求 | 状态 |
|--------|----------|------|
| GCC | 10+ | 完全支持 |
| Clang | 10+ | 完全支持 |
| MSVC | 19.28+ | 完全支持 |

编译选项：`-fcoroutines`（GCC < 10）、`/await`（MSVC）

---

## 相关参考

- [[boost-asio|Boost.Asio 协程]] — Asio 协程适配
- [[dev/coding/coroutine|协程约定]] — Prism 规范
- [[pmr-concepts|PMR 概念]] — 协程内存管理

---

## 进一步阅读

- C++ Coroutines: https://en.cppreference.com/w/cpp/language/coroutines
- Lewis Baker 的协程系列: https://lewissbaker.github.io/
- cppcoro 库: https://github.com/lewissbaker/cppcoro