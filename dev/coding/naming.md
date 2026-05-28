---
title: 命名规范
description: Prism 项目的命名约定，包括命名空间、文件、类型、函数等命名规则
layer: dev
tags: [dev, coding, naming]
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
namespace psm::instance { /* 代理核心模块 */ }
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

## 设计决策

### 为什么使用 `psm::` 命名空间前缀

`psm` 是 Prism 的缩写，作为根命名空间前缀有以下考量：

1. **避免冲突**: 全局命名空间污染是 C++ 项目的常见问题，短前缀 `psm::` 能有效隔离 Prism 的所有符号
2. **可读性**: 三字母前缀简短但可辨识，`psm::resolve`、`psm::stealth` 比 `prism::resolve` 更紧凑
3. **一致性**: 所有模块统一使用 `psm::` 前缀，代码中一眼即可区分项目内代码和第三方库代码
4. **嵌套清晰**: `psm::module::submodule` 的层级结构反映模块划分

### 为什么生产代码用 snake_case 而测试用 PascalCase

1. **生产代码 snake_case**: 遵循 C++ 标准库惯例（`std::unique_ptr`、`std::make_shared`），与标准库风格统一，降低认知负担
2. **测试代码 PascalCase**: 测试函数使用 `TestXxxYyy` 命名便于在测试输出中快速识别测试用例，也符合 Google Test / Catch2 等测试框架的惯例。通过风格差异可立即区分测试代码和被测代码
3. **双风格隔离**: 两种风格泾渭分明，避免了在代码审查时混淆测试函数和生产函数

### 为什么文件名用 snake_case

1. **跨平台兼容**: snake_case 文件名在所有操作系统（Windows、Linux、macOS）上大小写一致，避免因文件系统大小写敏感性差异导致的问题
2. **与类型名对应**: 文件名直接反映内部主要类型名（`connection_pool.hpp` → `class connection_pool`），查找方便
3. **C++ 生态惯例**: 大多数 C++ 项目（包括 Boost、LLVM）使用 snake_case 文件名

### 为什么标识符不超过 2 个词

1. **可读性**: 超过 2 个词的名称通常意味着功能不够聚焦，如 `process_inbound_tcp_packet_data` 应拆分为 `process_packet(data)`
2. **抽象层级**: 名称过长往往是抽象不足的信号，应通过引入新类型或函数来简化
3. **代码审查友好**: 短名称一行就能读完，减少代码审查的认知负担
4. **例外**: 允许最多 1 个下划线（即最多 2 个词），如 `connection_pool` 是两个词的合理组合

## 相关文档

- [[overview]] - 编码规范概览
- [[doxygen]] - Doxygen 注释规范
- [[dev/modules|modules]] - 模块命名空间说明

## 命名反例对比

以下列举 Prism 项目中常见的命名错误及其修正方案。

### 缩写滥用

| 错误命名 | 问题 | 正确命名 |
|----------|------|----------|
| `conn` | 过于简短，可读性差 | `connection` |
| `ctx` | 上下文含义不明 | `context` |
| `buf` | 缩写不一致 | `buffer` |
| `req` / `resp` | 不对称缩写 | `request` / `response` |
| `cfg` | 非标准缩写 | `config` |
| `tmp` | 语义模糊 | `temp_buffer` / `scratch_space` |

### 命名风格混用

**错误：**
```cpp
// 生产代码误用 PascalCase
class ConnectionPool { };        // 应为 connection_pool
struct DnsConfig { };            // 应为 dns_config
auto CreateConnection() -> ...;  // 应为 create_connection
```

**正确：**
```cpp
class connection_pool { };
struct dns_config { };
auto create_connection() -> ...;
```

### 变量名与类型名混淆

**错误：**
```cpp
// 变量使用类型风格
ConnectionPool Pool;             // 变量不应是大写开头
auto DNSConfig = parse_config(); // 变量不应是 PascalCase
```

**正确：**
```cpp
connection_pool pool;
auto dns_config = parse_config();
```

### 布尔值命名不表达意图

**错误：**
```cpp
bool flag;           // 什么标志？
bool state;          // 什么状态？
bool check;          // 是动词，像函数名
bool dns;            // 布尔值应该是"is_xxx"或"has_xxx"
```

**正确：**
```cpp
bool is_encrypted;
bool has_pending_request;
bool should_reconnect;
bool enable_dns_cache;
```

### 魔法数字命名

**错误：**
```cpp
std::vector<connection> connections;
connections.reserve(100);  // 100 是什么？
```

**正确：**
```cpp
constexpr size_t default_pool_capacity = 100;
std::vector<connection> connections;
connections.reserve(default_pool_capacity);
```

## 命名决策树

当为一个新元素命名时，遵循以下决策流程：

```
这是什么类型的元素？
│
├── 命名空间？
│   └── 使用 psm:: + 模块名（lowercase）
│       例：psm::resolve, psm::channel
│
├── 文件？
│   ├── 头文件？→ snake_case.hpp
│   └── 源文件？→ snake_case.cpp
│       例：connection_pool.hpp
│
├── 类型（class/struct/enum/enum class）？
│   └── snake_case
│       例：connection_pool, dns_config
│
├── 类型别名（using/typedef）？
│   └── snake_case + 语义后缀
│       例：connection_ptr, buffer_span
│
├── 函数/方法？
│   ├── 生产代码？→ snake_case 动词开头
│   │   例：create_connection, is_valid, parse_header
│   └── 测试代码？→ PascalCase TestXxx 格式
│       例：TestBasicGetRequest
│
├── 变量？
│   ├── 局部变量？→ snake_case 名词
│   │   例：connection_count, domain_name
│   ├── 成员变量？→ snake_case（可选 m_ 前缀）
│   │   例：m_connection_count, buffer_
│   └── 循环变量？→ 单字母或短名
│       例：i, it, pos
│
├── 常量？
│   ├── constexpr？→ snake_case 或 UPPER_SNAKE_CASE
│   │   例：max_buffer_size, MAX_RETRIES
│   └── const 局部？→ snake_case
│       例：default_timeout
│
└── 宏？（应尽量避免）
    └── UPPER_SNAKE_CASE
        例：PSM_ASSERT, PSM_UNUSED
```

### 函数命名动词前缀约定

| 前缀 | 语义 | 示例 |
|------|------|------|
| `create_` | 创建新对象 | `create_connection()` |
| `build_` | 构建复杂对象 | `build_packet()` |
| `parse_` | 解析数据 | `parse_clienthello()` |
| `is_` | 布尔查询 | `is_encrypted()` |
| `has_` | 是否存在 | `has_pending_data()` |
| `should_` | 是否应该 | `should_reconnect()` |
| `get_` | 获取（轻量操作） | `get_stats()` |
| `set_` | 设置 | `set_timeout()` |
| `handle_` | 处理事件 | `handle_packet()` |
| `process_` | 处理数据 | `process_buffer()` |

### `get_` 使用约束

`get_` 前缀仅用于**轻量级**操作，不应用于涉及 I/O、分配或复杂计算的函数：

**正确：**
```cpp
auto get_stats() const -> session_stats;       // 轻量，返回成员变量
auto get_capacity() const -> size_t;           // 轻量
```

**错误：**
```cpp
auto get_connection() -> net::awaitable<connection_ptr>;  // 涉及 I/O，应用 create_
auto get_all_data() -> std::vector<record>;               // 可能涉及分配和计算
```

## 常见命名错误

### 1. 匈牙利命名法残留

**问题：** 使用类型前缀标识变量类型，这是 C 时代的遗留习惯，在 C++ 中不需要。

```cpp
// 错误：匈牙利命名法
int iCount;
char* pszName;
std::vector<connection>* pvecConnections;

// 正确：语义命名
int connection_count;
std::string name;
std::vector<connection> connections;
```

### 2. 驼峰命名法混入

**问题：** 从其他语言或代码库迁移时带来的命名风格不一致。

```cpp
// 错误：来自其他项目的驼峰风格
class SessionManager { };
auto getConnection() -> connection_ptr;
int connectionCount;

// 正确：统一 snake_case
class session_manager { };
auto get_connection() -> connection_ptr;
int connection_count;
```

### 3. 双命名同义

**问题：** 同一概念在不同文件中使用不同名称。

```cpp
// 文件 A 中
using connection_ptr = std::shared_ptr<connection>;

// 文件 B 中
using conn_shared = std::shared_ptr<connection>;  // 不一致！

// 应统一为 connection_ptr
```

### 4. 过度泛化的名称

**问题：** 名称过于宽泛，无法表达具体用途。

```cpp
// 错误：过度泛化
class handler { };          // 处理什么？
auto process() -> void;     // 处理什么？
struct data { };            // 什么数据？

// 正确：具体化
class packet_handler { };
auto process_inbound_traffic() -> void;
struct dns_query_data { };
```

### 5. 否定式布尔名

**问题：** 使用否定前缀导致双重否定难以理解。

```cpp
// 错误：双重否定
bool not_found;     // !not_found 是什么？
bool no_error;      // !no_error 是什么？
bool dont_retry;    // !dont_retry 是什么？

// 正确：正向命名
bool is_missing;
bool has_error;
bool should_retry;
```

### 6. 缩写不一致

**问题：** 同一概念在不同地方使用不同缩写。

```cpp
// 错误：缩写不统一
auto init_connection() -> ...;    // init
auto initialize_pool() -> ...;    // initialize

// 正确：统一使用完整形式或统一缩写
auto initialize_connection() -> ...;
auto initialize_pool() -> ...;
```

### 7. 文件名与类型名不匹配

**问题：** 文件名与内部主要类型名称不对应，导致查找困难。

```cpp
// 错误：文件名与内容不匹配
// 文件：ConnectionPool.hpp（错误：生产代码不应 PascalCase）
// 内容：class connection_pool { };

// 正确
// 文件：connection_pool.hpp
// 内容：class connection_pool { };
```

## 命名审查自动化建议

以下 Clang-Tidy 检查可以帮助自动发现命名问题：

```yaml
# .clang-tidy 建议配置
Checks: >
  readability-identifier-naming,
  google-readability-avoid-underscore-in-googletest-name

CheckOptions:
  - key: readability-identifier-naming.ClassCase
    value: lower_case
  - key: readability-identifier-naming.StructCase
    value: lower_case
  - key: readability-identifier-naming.FunctionCase
    value: lower_case
  - key: readability-identifier-naming.VariableCase
    value: lower_case
  - key: readability-identifier-naming.ConstantCase
    value: UPPER_CASE
  - key: readability-identifier-naming.NamespaceCase
    value: lower_case
```