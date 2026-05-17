---
title: 命名规范
description: Prism 项目的命名约定，包括命名空间、文件、类型、函数等命名规则
---

# 命名规范

本文档定义 Prism 项目的命名约定，确保代码风格一致性和可读性。

## 命名空间

- **根命名空间**: `psm::`
- **子命名空间**: 根据模块功能划分

```cpp
namespace psm::resolve { /* DNS 解析模块 */ }
namespace psm::channel { /* 连接通道模块 */ }
namespace psm::recognition { /* 协议识别模块 */ }
namespace psm::agent { /* 代理核心模块 */ }
namespace psm::pipeline { /* 协议管道 */ }
namespace psm::multiplex { /* 多路复用 */ }
namespace psm::stealth { /* TLS 伪装 */ }
namespace psm::crypto { /* 加密模块 */ }
namespace psm::fault { /* 错误码 */ }
namespace psm::exception { /* 异常类型 */ }
namespace psm::memory { /* 内存管理 */ }
namespace psm::trace { /* 日志追踪 */ }
```

## 文件命名

- **风格**: snake_case
- **扩展名**: `.hpp` (头文件), `.cpp` (源文件)

```cpp
connection_pool.hpp    // 连接池
dns_upstream.hpp       // DNS 上游
session_manager.hpp    // 会话管理器
clienthello_analyzer.hpp  // ClientHello 分析器
```

## 类型命名

### 生产代码

**所有类型使用 snake_case**:

```cpp
// 类
class connection_pool { };
class session_manager { };

// 结构体
struct dns_config { };
struct upstream_result { };

// 枚举
enum class protocol_type { };
enum class cipher_suite { };

// 类型别名
using connection_ptr = std::shared_ptr<connection>;
using buffer_span = std::span<std::byte>;
```

### 测试代码

**测试函数使用 PascalCase**:

```cpp
void TestBasicGetRequest();
void TestDnsResolution();
void LogPass(const std::string& message);
void LogFail(const std::string& message);
```

## 函数命名

- **风格**: snake_case
- **返回类型**: 使用尾随返回类型

```cpp
// 正确示例
auto create_connection() -> net::awaitable<connection_ptr>;
auto resolve_domain(std::string_view domain) -> dns_result;
auto parse_clienthello(buffer_span data) -> clienthello_features;

// 使用 [[nodiscard]] 标记不可丢弃的返回值
[[nodiscard]] auto get_stats() const -> session_stats;
```

## 变量命名

- **局部变量**: snake_case
- **成员变量**: snake_case (可选 `m_` 或 `_` 前缀)
- **常量**: snake_case 或 UPPER_SNAKE_CASE

```cpp
// 局部变量
auto connection_count = 0;
std::string domain_name;

// 常量
constexpr size_t max_buffer_size = 8192;
constexpr auto default_timeout = std::chrono::seconds(30);
```

## 头文件保护

统一使用 `#pragma once`:

```cpp
#pragma once

namespace psm::module_name {
// ...
}
```

## 命名对照表

| 元素 | 风格 | 示例 |
|------|------|------|
| 命名空间 | lowercase | `psm::resolve` |
| 文件 | snake_case | `connection_pool.hpp` |
| 类 | snake_case | `session_manager` |
| 结构体 | snake_case | `dns_config` |
| 枚举/枚举类 | snake_case | `protocol_type` |
| 函数 | snake_case | `create_connection()` |
| 测试函数 | PascalCase | `TestBasicGetRequest()` |
| 局部变量 | snake_case | `connection_count` |
| 常量 | snake_case/UPPER | `max_buffer_size` |

## 相关文档

- [[overview]] - 编码规范概览
- [[doxygen]] - Doxygen 注释规范
- [[/dev/modules]] - 模块命名空间说明