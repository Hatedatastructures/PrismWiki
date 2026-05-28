---
tags: [exception, protocol]
layer: core
module: exception
source: I:/code/Prism/include/prism/exception/protocol.hpp
title: exception::protocol
---

# exception::protocol

协议配置异常，用于启动阶段的协议相关错误。运行时协议解析错误应使用 `fault::code`。

## 构造函数

| 构造方式 | 说明 |
|----------|------|
| `protocol(fault::code)` | 错误码构造（推荐） |
| `protocol(fault::code, string_view desc)` | 错误码 + 额外描述 |
| `protocol(const string&)` | 向后兼容，映射为 `generic_error` |
| `protocol(format_string, Args...)` | 格式化构造，映射为 `generic_error` |

`type_name()` 返回 `"PROTOCOL"`。

## 相关错误码

| 错误码 | 说明 |
|--------|------|
| `protocol_error` | 协议配置格式错误 |
| `parse_error` | 配置解析失败 |
| `not_supported` | 不支持的协议配置 |
| `socks5_authfail` | SOCKS5 认证配置无效 |

## 使用场景

| 场景 | 代码 |
|------|------|
| 协议配置不兼容 | `throw exception::protocol(fault::code::protocol_error, "版本不兼容");` |
| SOCKS5 认证配置 | `throw exception::protocol(fault::code::socks5_authfail);` |

## 约束

### 禁止在热路径使用

**类型**: 编码规范

**规则**: 运行时协议帧解析错误必须用 `fault::code`。

**源码依据**: `protocol.hpp:8-9`

## 引用关系

### 依赖

- [[core/exception/deviant|deviant]]：基类

### 被引用

- [[core/loader/load|loader]]：协议配置验证使用
