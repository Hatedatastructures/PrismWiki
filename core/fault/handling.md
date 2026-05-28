---
tags: [fault, handling]
layer: core
module: fault
source: I:/code/Prism/include/prism/fault/handling.hpp
title: fault::handling
---

# fault::handling

极简错误码检查适配层，提供对 `fault::code`、`std::error_code`、`boost::system::error_code` 的统一检查接口和双向转换。

## 核心接口

### 统一检查

```cpp
template <typename ErrorCode>
[[nodiscard]] constexpr auto succeeded(const ErrorCode &ec) noexcept -> bool;

template <typename ErrorCode>
[[nodiscard]] constexpr auto failed(const ErrorCode &ec) noexcept -> bool;
```

使用 `if constexpr` 编译时类型分发：`fault::code` 检查 `== success`，`std::error_code` / `boost::system::error_code` 检查 `!ec`。不支持的类型触发 `static_assert`。

### 转换

```cpp
[[nodiscard]] auto to_code(const boost::system::error_code &ec) noexcept -> code;
[[nodiscard]] auto to_code(const std::error_code &ec) noexcept -> code;
```

## 设计决策

### 为什么 to_code 先检查 category？

当 `error_code` 的 category 是 `"psm::fault"` 时（即源自 `fault::code` 的隐式转换），直接 `static_cast` 还原原始枚举值，无需逐个比较。这避免了"先转换出去再转换回来"的 N 次比较开销。

**后果**: 如果 category 不匹配，才进入逐个比较分支，未映射错误返回 `io_error`。

### 为什么 to_code(std::error_code) 用 static 预构造？

`std::make_error_code(std::errc)` 每次调用都构造临时对象。预构造为 `static const` 局部变量，只在首次进入函数时初始化一次，后续调用直接比较。

**后果**: 函数内有 7 个 `static const` 局部变量，但它们是 `std::error_code`（轻量对象），开销可忽略。

### 为什么 host_unreachable 映射到 `host_noreply` 而非新增枚举？

`host_noreply` 和 `net_noreply` 是项目内部对"不可达"的命名（遵循 2 词命名规则），语义分别对应 POSIX 的 `EHOSTUNREACH` 和 `ENETUNREACH`。

**后果**: wiki 文档中列出的枚举名（`host_noreply`）与 POSIX 名称（`host_unreachable`）不同，需注意对应关系。

## 约束

### 热路径禁止异常

**类型**: 编码规范

**规则**: 热路径（网络 I/O、协议解析、TLS 握手）中所有错误必须通过 `fault::code` 传播，禁止使用异常。

**违反后果**: 异常栈展开导致延迟不可预测，破坏热路径性能保证。

**源码依据**: `handling.hpp:11`

### 未映射错误丢失信息

**类型**: 信息丢失

**规则**: `to_code()` 中未映射的外部错误码统一返回 `io_error`。

**违反后果**: 原始错误信息丢失。如果需要保留原始错误，应在调用 `to_code()` 前记录日志。

**源码依据**: `handling.hpp:140,207`

## Boost.Asio 错误码映射

| Boost 错误 | fault::code |
|------------|-------------|
| `eof` | `eof` |
| `operation_aborted` | `canceled` |
| `timed_out` | `timeout` |
| `connection_refused` | `connection_refused` |
| `connection_reset` | `connection_reset` |
| `connection_aborted` | `connection_aborted` |
| `host_unreachable` | `host_noreply` |
| `network_unreachable` | `net_noreply` |
| `no_buffer_space` | `resource_unavailable` |
| 其他 | `io_error` |

## 错误处理最佳实践

| 规则 | 说明 |
|------|------|
| 透传错误码 | `return ec;`，不要包装为 `generic_error` |
| 日志加上下文 | `trace::error("conn failed: {}, host={}", describe(ec), host);` |
| 不吞错误 | 不要忽略 `failed()` 返回值 |

## 引用关系

### 依赖

- [[core/fault/code|fault::code]]：错误码枚举
- [[core/fault/compatible|fault::compatible]]：category() 和 make_error_code()

### 被引用

- [[core/fault/overview|Fault 模块]]：总览
- 几乎所有模块的异步回调中使用 `to_code()` 转换
