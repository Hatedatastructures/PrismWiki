---
title: Error Handling Strategy
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [programming, cpp, error, exception, error-code]
---

# 错误处理策略

> C++ 错误处理方式：异常 vs 错误码、Result 类型、Prism 策略。
> 最后更新：2026-05-17

---

## 错误处理方式

### 异常

```cpp
// 异常方式
void process() {
    if (error_condition) {
        throw std::runtime_error("error message");
    }
}

try {
    process();
} catch (const std::exception& e) {
    handle_error(e.what());
}
```

### 错误码

```cpp
// 错误码方式
int process() {
    if (error_condition) {
        return -1;  // 或特定错误码
    }
    return 0;  // 成功
}

int result = process();
if (result != 0) {
    handle_error(result);
}
```

---

## 异常优缺点

### 优点

| 特点 | 说明 |
|------|------|
| 自动传播 | 错误自动向上传播 |
| 类型安全 | 不同异常类型区分 |
| 不可忽略 | 未捕获异常终止程序 |
| 信息丰富 | 携带消息、堆栈 |

### 缺点

| 特点 | 说明 |
|------|------|
| 性能开销 | 抛出/捕获有成本 |
| 隐式流程 | 控制流不明显 |
| 二进制代价 | 异常表增加代码大小 |
| 不适合热路径 | 高频场景影响性能 |

---

## 错误码优缺点

### 优点

| 特点 | 说明 |
|------|------|
| 显式流程 | 控制流清晰 |
| 零开销 | 无异常机制成本 |
| 适合热路径 | 高频场景首选 |
| C 兼容 | 跨语言友好 |

### 缺点

| 特点 | 说明 |
|------|------|
| 可忽略 | 返回值可能被忽略 |
| 传播繁琐 | 需手动向上传递 |
| 信息有限 | 整数码信息少 |
| 类型单一 | 需额外机制区分 |

---

## Result 类型

### std::expected（C++23）

```cpp
#include <expected>

std::expected<int, std::string> process() {
    if (error_condition) {
        return std::unexpected("error message");
    }
    return 42;
}

auto result = process();
if (result) {
    int value = *result;
} else {
    std::string error = result.error();
}
```

### 自定义 Result

```cpp
template<typename T, typename E>
class Result {
    std::variant<T, E> data_;
public:
    bool is_ok() const { return std::holds_alternative<T>(data_); }
    T& value() & { return std::get<T>(data_); }
    E& error() & { return std::get<E>(data_); }
    
    // monadic 操作
    template<typename F>
    auto map(F f) -> Result<decltype(f(std::declval<T>())), E> {
        if (is_ok()) return f(value());
        return error();
    }
};
```

---

## Prism 错误处理策略

### 分层策略

| 层级 | 方式 | 理由 |
|------|------|------|
| 热路径 | 错误码/Result | 性能优先 |
| 边界层 | 异常 | 简化传播 |
| 用户层 | 异常 | 信息丰富 |

详见 [[dev/coding/error|错误处理策略]]。

### 具体场景

```cpp
// 热路径：错误码
ErrorCode read_data(tcp::socket& socket, std::span<uint8_t> buffer) {
    asio::error_code ec;
    size_t n = socket.read_some(asio::buffer(buffer), ec);
    if (ec) return ErrorCode::NetworkError;
    return ErrorCode::Success;
}

// 协程边界：异常
asio::awaitable<void> handle_session(tcp::socket socket) {
    try {
        co_await process_session(socket);
    } catch (const std::exception& e) {
        log::error("session error: {}", e.what());
    }
}

// 配置加载：异常
Config load_config(const std::string& path) {
    if (!file_exists(path)) {
        throw ConfigError("config file not found: " + path);
    }
    // ...
}
```

---

## 错误码系统

### Prism 错误码

```cpp
enum class ErrorCode : int {
    Success = 0,
    
    // 网络错误
    NetworkError = 100,
    ConnectionClosed = 101,
    TimeoutError = 102,
    
    // 协议错误
    ProtocolError = 200,
    InvalidHeader = 201,
    InvalidPayload = 202,
    
    // 加密错误
    CryptoError = 300,
    DecryptFailed = 301,
    InvalidKey = 302,
    
    // 配置错误
    ConfigError = 400,
    InvalidConfig = 401,
};
```

详见 [[core/fault/overview|Fault 模块]]。

---

## 异常体系

### Prism 异常层次

```cpp
// 基类
class PrismException : public std::runtime_error {
public:
    explicit PrismException(const std::string& msg);
    ErrorCode code() const noexcept;
};

// 网络异常
class NetworkException : public PrismException {
public:
    NetworkException(ErrorCode code, const std::string& msg);
};

// 协议异常
class ProtocolException : public PrismException {
public:
    ProtocolException(ErrorCode code, const std::string& msg);
};
```

详见 [[core/exception/overview|Exception 模块]]。

---

## 协程错误处理

### try-catch 模式

```cpp
asio::awaitable<void> safe_handler(tcp::socket socket) {
    try {
        co_await process(socket);
    } catch (const NetworkException& e) {
        log::error("network: {}", e.what());
        socket.close();
    } catch (const ProtocolException& e) {
        log::error("protocol: {}", e.what());
        socket.close();
    } catch (const std::exception& e) {
        log::error("unknown: {}", e.what());
        socket.close();
    }
}
```

### error_code 模式

```cpp
asio::awaitable<ErrorCode> read_with_ec(tcp::socket& socket) {
    asio::error_code ec;
    co_await socket.async_read_some(asio::buffer(data), 
                                     asio::redirect_error(asio::use_awaitable, ec));
    if (ec) {
        co_return ErrorCode::NetworkError;
    }
    co_return ErrorCode::Success;
}
```

---

## 相关参考

- [[core/fault/overview|Fault 模块]] — 错误码定义
- [[core/exception/overview|Exception 模块]] — 异常体系
- [[dev/coding/error|错误处理策略]] — Prism 规范

---

## 进一步阅读

- std::expected: https://en.cppreference.com/w/cpp/utility/expected
- 异常处理: https://en.cppreference.com/w/cpp/language/exceptions