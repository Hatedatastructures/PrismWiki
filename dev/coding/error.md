---
title: 错误处理
description: Prism 双轨错误处理策略，热路径错误码与启动异常分离
---

# 错误处理

Prism 采用**双轨错误处理策略**，根据执行阶段选择合适的错误处理机制。

## 双轨策略

| 执行阶段 | 错误处理方式 | 原因 |
|----------|--------------|------|
| 热路径 | `fault::code` 错误码 | 性能优先，避免异常开销 |
| 启动/致命错误 | 异常层次结构 | 配置错误需要明确报告 |

## Fault 模块

热路径使用 `fault::code` 枚举返回错误码，不抛异常：

### 模块结构

```
include/prism/fault/
├── code.hpp        // 错误码枚举
├── handling.hpp    // 错误处理
└── compatible.hpp  // 兼容性处理
```

### 错误码枚举

```cpp
namespace psm::fault {
    enum class code {
        success = 0,

        // 网络错误
        connection_refused,
        connection_timeout,
        connection_reset,

        // 协议错误
        invalid_protocol,
        malformed_packet,
        unsupported_version,

        // DNS 错误
        dns_resolution_failed,
        dns_timeout,

        // 其他错误
        buffer_overflow,
        invalid_argument,
        // ...
    };
}
```

### 使用示例

```cpp
auto read_packet(tcp::socket& socket) -> net::awaitable<result<packet, fault::code>> {
    std::array<std::byte, 1024> buffer;
    auto ec = std::error_code{};

    size_t n = co_await socket.async_read_some(
        net::buffer(buffer),
        net::redirect_error(net::use_awaitable, ec)
    );

    if (ec) {
        co_return fault::code::connection_reset;
    }

    if (n < header_size) {
        co_return fault::code::malformed_packet;
    }

    co_return parse_packet(buffer.data(), n);
}
```

### 错误处理函数

```cpp
namespace psm::fault {
    // 将错误码转换为字符串
    auto to_string(code c) -> std::string_view;

    // 将错误码转换为 Boost.Asio 错误码
    auto to_error_code(code c) -> std::error_code;

    // 检查是否为可恢复错误
    auto is_recoverable(code c) -> bool;
}
```

## Exception 模块

启动和致命错误使用异常层次结构：

### 模块结构

```
include/prism/exception/
├── deviant.hpp     // 基类异常
├── network.hpp     // 网络异常
├── protocol.hpp     // 协议异常
└── security.hpp     // 安全异常
```

### 异常层次

```
exception::deviant (基类)
├── exception::network   // 网络错误
├── exception::protocol  // 协议错误
└── exception::security  // 安全错误
```

### 基类定义

```cpp
namespace psm::exception {
    class deviant : public std::exception {
    public:
        deviant(std::string message, std::source_location loc = std::source_location::current());

        auto what() const noexcept -> const char* override;
        auto where() const noexcept -> const std::source_location&;

    private:
        std::string message_;
        std::source_location location_;
    };
}
```

### 派生异常

```cpp
namespace psm::exception {
    // 网络异常
    class network : public deviant {
    public:
        network(std::string message, std::source_location loc = {});
        // bind_failed, listen_failed, accept_failed...
    };

    // 协议异常
    class protocol : public deviant {
    public:
        protocol(std::string message, std::source_location loc = {});
        // invalid_handshake, unsupported_version...
    };

    // 安全异常
    class security : public deviant {
    public:
        security(std::string message, std::source_location loc = {});
        // certificate_error, authentication_failed...
    };
}
```

### 使用场景

#### 配置加载错误

```cpp
auto load_config(const std::string& path) -> config {
    std::ifstream file(path);
    if (!file) {
        throw exception::network(
            fmt::format("Failed to open config file: {}", path)
        );
    }

    try {
        return parse_config(file);
    } catch (const glaze::error& e) {
        throw exception::protocol(
            fmt::format("Invalid config format: {}", e.what())
        );
    }
}
```

#### TLS 证书错误

```cpp
auto load_certificate(const std::string& path) -> certificate {
    if (!std::filesystem::exists(path)) {
        throw exception::security(
            fmt::format("Certificate file not found: {}", path)
        );
    }
    // ...
}
```

## 错误处理指南

### 热路径原则

1. **不抛异常** - 使用 `fault::code` 返回错误
2. **避免异常开销** - 异常处理有运行时成本
3. **明确错误类型** - 使用具体的错误码

```cpp
// 热路径示例
auto handle_request(request& req) -> net::awaitable<result<response, fault::code>> {
    if (!validate(req)) {
        co_return fault::code::invalid_argument;
    }

    auto conn = co_await get_connection();
    if (!conn) {
        co_return fault::code::connection_refused;
    }

    co_return co_await process(conn, req);
}
```

### 启动阶段原则

1. **使用异常** - 配置错误、初始化失败
2. **明确错误位置** - 使用 `std::source_location`
3. **提供清晰信息** - 错误消息包含上下文

```cpp
// 启动阶段示例
void initialize_server(const config& cfg) {
    if (cfg.port < 1 || cfg.port > 65535) {
        throw exception::deviant(
            fmt::format("Invalid port: {}", cfg.port)
        );
    }

    if (!load_certificate(cfg.cert_path)) {
        throw exception::security("Failed to load certificate");
    }
}
```

## 错误传播

### 协程中的错误传播

```cpp
auto operation() -> net::awaitable<result<data, fault::code>> {
    auto r1 = co_await step1();
    if (!r1) {
        co_return r1.error();  // 传播错误
    }

    auto r2 = co_await step2(r1.value());
    if (!r2) {
        co_return r2.error();  // 传播错误
    }

    co_return r2.value();
}
```

### 异常捕获边界

在协程边界捕获异常并转换为错误码：

```cpp
auto safe_operation() -> net::awaitable<result<data, fault::code>> {
    try {
        co_return co_await risky_operation();
    } catch (const exception::network& e) {
        log_error("Network error: {}", e.what());
        co_return fault::code::connection_refused;
    } catch (const std::exception& e) {
        log_error("Unexpected error: {}", e.what());
        co_return fault::code::internal_error;
    }
}
```

## 相关文档

- [[overview]] - 编码规范概览
- [[coroutine]] - 协程约定
- [[dev/modules|modules]] - 模块结构