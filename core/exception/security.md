---
tags: [exception, security]
layer: core
module: exception
source: include/prism/exception/security.hpp
title: exception::security
---

# exception::security

安全配置异常，用于启动阶段的安全层错误。运行时认证/授权失败应使用 `fault::code`。

## 构造函数

| 构造方式 | 说明 |
|----------|------|
| `security(fault::code)` | 错误码构造（推荐） |
| `security(fault::code, string_view desc)` | 错误码 + 额外描述 |
| `security(const string&)` | 向后兼容，映射为 `generic_error` |
| `security(format_string, Args...)` | 格式化构造，映射为 `generic_error` |

`type_name()` 返回 `"SECURITY"`。

## 相关错误码

| 错误码 | 说明 |
|--------|------|
| `certfail` | SSL 证书加载失败 |
| `keyfail` | SSL 密钥加载失败 |
| `verifyfail` | 证书验证失败 |
| `config_err` | 安全配置格式错误 |
| `file_openfail` | 配置文件打开失败 |

## 使用场景

| 场景 | 代码 |
|------|------|
| 证书加载失败 | `throw exception::security(fault::code::certfail, "路径无效");` |
| 密钥加载失败 | `throw exception::security(fault::code::keyfail);` |
| 配置文件不存在 | `throw exception::security(fault::code::file_openfail, path);` |

## 约束

### 禁止在热路径使用

**类型**: 编码规范

**规则**: 运行时认证失败（如密码错误、token 过期）必须用 `fault::code::auth_failed`，不能用此异常。

**违反后果**: 异常栈展开破坏协程执行流，且安全回调中构造异常可能导致二次分配失败。

**源码依据**: `security.hpp:8-9`

## 故障场景

### 安全异常触发链

**触发条件**: TLS 证书路径无效、密钥格式错误、配置文件缺失。

**传播路径**: `loader::load()` 解析配置 → 发现证书路径不存在 → `throw exception::security(certfail, path)` → `main()` 的 `try/catch` 捕获 → 打印 `dump()` → 进程退出。

**外部表现**: 启动失败，日志输出 `[loader.cpp:38] [SECURITY:26] certfail: /etc/prism/server.crt`。

**恢复机制**: 需要人工修复配置后重启。

**日志关键字**: `[SECURITY:` 前缀

## 引用关系

### 依赖

- [[core/exception/deviant|deviant]]：基类

### 被引用

- [[core/loader/load|loader]]：配置加载验证
- [[core/instance/worker/tls|worker/tls]]：TLS 初始化
