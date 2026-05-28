---
tags: [fault, compatible]
layer: core
module: fault
source: I:/code/Prism/include/prism/fault/compatible.hpp
title: fault::compatible
---

# fault::compatible

错误码标准库兼容性支持，提供 `fault::code` 与 `std::error_code` 和 `boost::system::error_code` 的双向兼容。

## 核心组件

| 组件 | 说明 |
|------|------|
| `fault_category` | `std::error_category` 实现，name() 返回 `"psm::fault"` |
| `category()` | 全局单例引用（Meyer's singleton，线程安全） |
| `make_error_code(code)` | fault::code → std::error_code 转换 |
| `cached_message(code)` | 缓存的错误消息引用（首次分配后零开销） |

## 设计决策

### 为什么需要 compatible 层？

Boost.Asio 的异步操作回调使用 `boost::system::error_code`，标准库 API 使用 `std::error_code`。`fault::code` 是项目内部的统一错误表示，需要与两个外部错误码系统双向转换。兼容层通过 `is_error_code_enum` 特化实现隐式转换，避免调用方每次手动包装。

**后果**: 修改此文件可能影响 ABI 兼容性。新增 `std`/`boost::system` 命名空间特化需谨慎。

### 为什么 cached_message() 返回 `const string&` 而非 `string_view`？

`std::error_category::message()` 签名要求返回 `std::string`。`cached_message()` 首次调用时将所有 `describe()` 结果预构造为 `std::string` 并缓存在静态数组中，后续返回引用无需分配。

**后果**: 首次调用有一次批量分配（64 个 string），后续调用零开销。缓存在静态析构阶段可能失效，但此时不应再调用。

### 为什么同时特化 std 和 boost::system 命名空间？

项目同时使用 `std::error_code`（标准库 API）和 `boost::system::error_code`（Boost.Asio），两个类型是独立的，各自需要独立的 `is_error_code_enum` 特化才能支持隐式转换。

**后果**: 两个命名空间的 `fault_category` 是独立实例，但行为一致（都调用 `cached_message()`）。

## 约束

### 不要在静态析构阶段使用 category()

**类型**: 生命周期

**规则**: `category()` 返回的引用在静态析构阶段可能已失效。

**违反后果**: 访问已析构的静态对象，未定义行为。

**源码依据**: `compatible.hpp:95-100`

## 使用场景

| 场景 | 代码 |
|------|------|
| 隐式转换 | `std::error_code ec = fault::code::timeout;` |
| 无序容器 | `std::unordered_set<fault::code> errors;` |
| Boost 转换 | `boost::system::error_code bec = fault::code::auth_failed;` |

## 引用关系

### 依赖

- [[core/fault/code|fault::code]]：错误码枚举

### 被引用

- [[core/fault/handling|handling]]：使用 category() 反向转换
- [[core/fault/overview|Fault 模块]]：总览
