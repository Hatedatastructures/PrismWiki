---
tags: [exception, network]
layer: core
module: exception
source: I:/code/Prism/include/prism/exception/network.hpp
title: exception::network
---

# exception::network

网络配置异常，用于启动阶段的网络相关错误。运行时网络 I/O 错误应使用 `fault::code`。

## 构造函数

| 构造方式 | 说明 |
|----------|------|
| `network(fault::code)` | 错误码构造（推荐） |
| `network(fault::code, string_view desc)` | 错误码 + 额外描述 |
| `network(const string&)` | 向后兼容，映射为 `generic_error` |
| `network(format_string, Args...)` | 格式化构造，映射为 `generic_error` |

所有构造函数自动捕获 `std::source_location`。`type_name()` 返回 `"NETWORK"`。

## 相关错误码

| 错误码 | 说明 |
|--------|------|
| `dns_failed` | DNS 配置错误 |
| `unreachable` | 上游服务器不可达 |
| `port_busy` | 端口已被占用 |
| `config_err` | 网络配置格式错误 |

## 使用场景

| 场景 | 代码 |
|------|------|
| 端口绑定失败 | `throw exception::network(fault::code::port_busy, "8080 已占用");` |
| DNS 配置无效 | `throw exception::network(fault::code::dns_failed);` |
| 上游配置无效 | `throw exception::network(fault::code::unreachable);` |

## 约束

### 禁止在热路径使用

**类型**: 编码规范

**规则**: 异步 I/O 回调中的网络错误必须用 `fault::code`，不能用此异常。

**违反后果**: 异常栈展开破坏协程执行流，延迟不可预测。

**源码依据**: `network.hpp:8-9`

## 引用关系

### 依赖

- [[core/exception/deviant|deviant]]：基类

### 被引用

- [[core/loader/load|loader]]：配置加载使用
- [[core/instance/listener|listener]]：启动绑定使用
