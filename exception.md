---
title: "Exception 模块"
module: "exception"
type: module
tags: [exception, 异常, deviant, network, protocol, security]
created: 2026-05-15
updated: 2026-05-15
related:
  - fault/code
  - fault/compatible
  - agent/config
  - loader/load
  - agent/worker/tls
---

# Exception 模块

> 模块: [[exception|exception]]

## 概述

Exception 模块定义异常层次结构。基于 `deviant` 基类，派生出 network、protocol、security 三类异常。遵循热路径无异常原则，异常仅用于启动阶段配置错误或致命错误，热路径应使用 `fault::code` 错误码。

---

## deviant.hpp — 异常基类

> 源码: `include/prism/exception/deviant.hpp`
> 模块: [[exception|exception]]

### 概述

异常基类定义。所有自定义异常的公共基类，支持源位置捕获和格式化消息。基于 `std::error_code` 架构，提供结构化异常信息。该类是抽象基类，强制子类提供类型分类。

### 命名空间

`psm::exception`

### 类: deviant

- **功能说明**: 项目异常基类。继承自 `std::runtime_error`，核心存储 `std::error_code`，自动捕获抛出点的位置信息，并支持格式化消息。
- **签名**: `class deviant : public std::runtime_error`

#### deviant::deviant(error_code, desc, loc)

- **功能说明**: 主构造函数（错误码 + 可选描述）。使用错误码和可选描述构造异常，自动捕获抛出点的源码位置。这是异常构造的首选方式。
- **签名**: `explicit deviant(std::error_code ec, std::string_view desc = {}, const std::source_location &loc = std::source_location::current())`
- **参数**: `ec` — 错误码；`desc` — 可选描述信息；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `create_what()`
- **被调用（向上）**: `network::network(code)`、`protocol::protocol(code)`、`security::security(code)`
- **知识域**: 结构化异常构造

#### deviant::deviant(string, loc)

- **功能说明**: 构造函数（向后兼容字符串）。将字符串转换为 `generic_error` 错误码，建议迁移到错误码构造函数。
- **签名**: `explicit deviant(const std::string &msg, const std::source_location &loc = std::source_location::current())`
- **参数**: `msg` — 错误消息；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant(error_code, desc, loc)`、`fault::make_error_code()`
- **被调用（向上）**: `network::network(string)`、`protocol::protocol(string)`、`security::security(string)`
- **知识域**: 向后兼容异常构造

#### deviant::deviant(source_location, format_string, args...)

- **功能说明**: 构造函数（带格式化参数）。将格式化字符串转换为 `generic_error` 错误码，建议迁移到错误码构造函数。
- **签名**: `template <typename... Args> explicit deviant(const std::source_location &loc, std::format_string<Args...> fmt, Args &&...args)`
- **参数**: `loc` — 源码位置；`fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant(string, loc)`、`std::format()`
- **被调用（向上）**: `network::network(loc, fmt, args...)`、`protocol::protocol(loc, fmt, args...)`、`security::security(fmt, args...)`
- **知识域**: 格式化异常构造

#### deviant::error_code()

- **功能说明**: 获取异常的错误码。
- **签名**: `[[nodiscard]] const std::error_code &error_code() const noexcept`
- **参数**: 无
- **返回值**: `const std::error_code&` — 错误码引用
- **调用（向下）**: 无
- **被调用（向上）**: 异常处理流程中的错误码提取
- **知识域**: 异常错误码访问

#### deviant::location()

- **功能说明**: 获取异常抛出时的位置信息。
- **签名**: `[[nodiscard]] const std::source_location &location() const noexcept`
- **参数**: 无
- **返回值**: `const std::source_location&` — 包含文件名、行号、列号等位置信息
- **调用（向下）**: 无
- **被调用（向上）**: `dump()`、异常诊断
- **知识域**: 异常源位置访问

#### deviant::filename()

- **功能说明**: 获取异常抛出时的文件名（不包含路径）。
- **签名**: `[[nodiscard]] std::string filename() const`
- **参数**: 无
- **返回值**: `std::string` — 文件名
- **调用（向下）**: `std::filesystem::path::filename()`
- **被调用（向上）**: `dump()`
- **知识域**: 异常文件名提取

#### deviant::dump()

- **功能说明**: 格式化异常信息。生成包含文件名、行号、异常类型、错误码和错误描述的详细字符串。输出格式为 `[filename:line] [TYPE:value] description`。
- **签名**: `[[nodiscard]] virtual std::string dump() const`
- **参数**: 无
- **返回值**: `std::string` — 格式化后的异常信息
- **调用（向下）**: `filename()`、`type_name()`、`std::runtime_error::what()`
- **被调用（向上）**: 异常日志记录、调试输出
- **知识域**: 异常格式化输出

#### deviant::type_name() [protected, pure virtual]

- **功能说明**: 获取异常类型名称。子类必须实现此方法，返回自己的类型标识符。
- **签名**: `[[nodiscard]] virtual std::string_view type_name() const noexcept = 0`
- **参数**: 无
- **返回值**: `std::string_view` — 异常类型名称
- **调用（向下）**: 无
- **被调用（向上）**: `dump()`
- **知识域**: 异常类型分类

#### deviant::create_what() [private, static]

- **功能说明**: 构建 `what()` 字符串。如果有额外描述则组合错误码消息和描述，否则仅返回错误码消息。
- **签名**: `static std::string create_what(const std::error_code &ec, std::string_view desc)`
- **参数**: `ec` — 错误码；`desc` — 额外描述
- **返回值**: `std::string` — 组合后的错误消息
- **调用（向下）**: `std::error_code::message()`
- **被调用（向上）**: `deviant::deviant(error_code, desc, loc)`
- **知识域**: 异常消息构建

---

## network.hpp — 网络异常

> 源码: `include/prism/exception/network.hpp`
> 模块: [[exception|exception]]

### 概述

网络异常子类。用于处理网络配置和初始化阶段的错误。运行时网络 I/O 错误应使用错误码而非异常。

### 类: network

- **功能说明**: 网络异常。表示网络层相关的异常情况，继承自 `exception::deviant`。类型名称为 `NETWORK`。
- **签名**: `class network : public deviant`

#### network::network(code, loc)

- **功能说明**: 构造函数（网络错误码）。
- **签名**: `explicit network(fault::code err, const std::source_location &loc = std::source_location::current())`
- **参数**: `err` — 网络错误码；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(error_code, {}, loc)`、`fault::make_error_code()`
- **被调用（向上）**: `loader::load()` 解析失败
- **知识域**: 网络异常构造（错误码）

#### network::network(code, desc, loc)

- **功能说明**: 构造函数（网络错误码 + 额外描述）。
- **签名**: `explicit network(fault::code err, std::string_view desc, const std::source_location &loc = std::source_location::current())`
- **参数**: `err` — 网络错误码；`desc` — 额外描述信息；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(error_code, desc, loc)`、`fault::make_error_code()`
- **被调用（向上）**: 网络初始化错误
- **知识域**: 网络异常构造（带描述）

#### network::network(string, loc)

- **功能说明**: 构造函数（向后兼容字符串）。将字符串转换为 `generic_error` 错误码。
- **签名**: `explicit network(const std::string &msg, const std::source_location &loc = std::source_location::current())`
- **参数**: `msg` — 错误消息；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(string, loc)`
- **被调用（向上）**: 向后兼容场景
- **知识域**: 向后兼容网络异常构造

#### network::network(format_string, args...)

- **功能说明**: 构造函数（带格式化参数，自动获取位置）。
- **签名**: `template <typename... Args> explicit network(std::format_string<Args...> fmt, Args &&...args)`
- **参数**: `fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(loc, fmt, args...)`
- **被调用（向上）**: 格式化错误消息场景
- **知识域**: 格式化网络异常构造

#### network::network(source_location, format_string, args...)

- **功能说明**: 构造函数（带格式化参数，指定位置）。
- **签名**: `template <typename... Args> explicit network(const std::source_location &loc, std::format_string<Args...> fmt, Args &&...args)`
- **参数**: `loc` — 源码位置；`fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(loc, fmt, args...)`
- **被调用（向上）**: 需要显式指定源位置的场景
- **知识域**: 指定位置的网络异常构造

#### network::type_name() [protected]

- **功能说明**: 获取异常类型名称。返回固定值 `NETWORK`。
- **签名**: `[[nodiscard]] std::string_view type_name() const noexcept override`
- **参数**: 无
- **返回值**: `std::string_view` — 固定为 `"NETWORK"`
- **调用（向下）**: 无
- **被调用（向上）**: `deviant::dump()`
- **知识域**: 网络异常类型标识

---

## protocol.hpp — 协议异常

> 源码: `include/prism/exception/protocol.hpp`
> 模块: [[exception|exception]]

### 概述

协议异常子类。用于处理协议解析、握手、格式验证等协议层错误。协议异常应主要用于协议栈的初始化和验证阶段。

### 类: protocol

- **功能说明**: 协议异常。表示协议层相关的异常情况，继承自 `exception::deviant`。类型名称为 `PROTOCOL`。
- **签名**: `class protocol : public deviant`

#### protocol::protocol(code, loc)

- **功能说明**: 构造函数（协议错误码）。使用 `fault::code` 错误码构造协议异常，自动捕获调用点的源码位置。
- **签名**: `explicit protocol(psm::fault::code err, const std::source_location &loc = std::source_location::current())`
- **参数**: `err` — 协议错误码；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(error_code, {}, loc)`、`fault::make_error_code()`
- **被调用（向上）**: TLS 配置加载失败
- **知识域**: 协议异常构造（错误码）

#### protocol::protocol(code, desc, loc)

- **功能说明**: 构造函数（协议错误码 + 额外描述）。
- **签名**: `explicit protocol(psm::fault::code err, std::string_view desc, const std::source_location &loc = std::source_location::current())`
- **参数**: `err` — 协议错误码；`desc` — 额外描述信息；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(error_code, desc, loc)`、`fault::make_error_code()`
- **被调用（向上）**: 协议握手/协商失败
- **知识域**: 协议异常构造（带描述）

#### protocol::protocol(string, loc)

- **功能说明**: 构造函数（向后兼容字符串）。将字符串转换为 `generic_error` 错误码。
- **签名**: `explicit protocol(const std::string &msg, const std::source_location &loc = std::source_location::current())`
- **参数**: `msg` — 错误消息；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(string, loc)`
- **被调用（向上）**: 向后兼容场景
- **知识域**: 向后兼容协议异常构造

#### protocol::protocol(format_string, args...)

- **功能说明**: 构造函数（带格式化参数，自动获取位置）。
- **签名**: `template <typename... Args> explicit protocol(std::format_string<Args...> fmt, Args &&...args)`
- **参数**: `fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(loc, fmt, args...)`
- **被调用（向上）**: TLS 证书/密钥加载失败
- **知识域**: 格式化协议异常构造

#### protocol::protocol(source_location, format_string, args...)

- **功能说明**: 构造函数（带格式化参数，指定位置）。
- **签名**: `template <typename... Args> explicit protocol(const std::source_location &loc, std::format_string<Args...> fmt, Args &&...args)`
- **参数**: `loc` — 源码位置；`fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(loc, fmt, args...)`
- **被调用（向上）**: 需要显式指定源位置的场景
- **知识域**: 指定位置的协议异常构造

#### protocol::type_name() [protected]

- **功能说明**: 获取异常类型名称。返回固定值 `PROTOCOL`。
- **签名**: `[[nodiscard]] std::string_view type_name() const noexcept override`
- **参数**: 无
- **返回值**: `std::string_view` — 固定为 `"PROTOCOL"`
- **调用（向下）**: 无
- **被调用（向上）**: `deviant::dump()`
- **知识域**: 协议异常类型标识

---

## security.hpp — 安全异常

> 源码: `include/prism/exception/security.hpp`
> 模块: [[exception|exception]]

### 概述

安全异常子类。用于处理安全相关的错误，如认证失败、授权拒绝、证书无效等。仅用于启动阶段安全配置错误，运行时安全错误应使用错误码。

### 类: security

- **功能说明**: 安全异常。表示安全层相关的异常情况，继承自 `exception::deviant`。类型名称为 `SECURITY`。
- **签名**: `class security : public deviant`

#### security::security(code, loc)

- **功能说明**: 构造函数（安全错误码）。
- **签名**: `explicit security(psm::fault::code err, const std::source_location &loc = std::source_location::current())`
- **参数**: `err` — 安全错误码；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(error_code, {}, loc)`、`fault::make_error_code()`
- **被调用（向上）**: 安全配置初始化错误
- **知识域**: 安全异常构造（错误码）

#### security::security(code, desc, loc)

- **功能说明**: 构造函数（安全错误码 + 额外描述）。
- **签名**: `explicit security(psm::fault::code err, std::string_view desc, const std::source_location &loc = std::source_location::current())`
- **参数**: `err` — 安全错误码；`desc` — 额外描述信息；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(error_code, desc, loc)`、`fault::make_error_code()`
- **被调用（向上）**: 安全验证失败
- **知识域**: 安全异常构造（带描述）

#### security::security(string, loc)

- **功能说明**: 构造函数（向后兼容字符串）。将字符串转换为 `generic_error` 错误码。
- **签名**: `explicit security(const std::string &msg, const std::source_location &loc = std::source_location::current())`
- **参数**: `msg` — 错误消息；`loc` — 源码位置（默认自动获取）
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(string, loc)`
- **被调用（向上）**: `main.cpp` 启动流程
- **知识域**: 向后兼容安全异常构造

#### security::security(format_string, args...)

- **功能说明**: 构造函数（带格式化参数，自动获取位置）。
- **签名**: `template <typename... Args> explicit security(std::format_string<Args...> fmt, Args &&...args)`
- **参数**: `fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(loc, fmt, args...)`
- **被调用（向上）**: 文件打开失败、配置错误
- **知识域**: 格式化安全异常构造

#### security::security(source_location, format_string, args...)

- **功能说明**: 构造函数（带格式化参数，指定位置）。
- **签名**: `template <typename... Args> explicit security(const std::source_location &loc, std::format_string<Args...> fmt, Args &&...args)`
- **参数**: `loc` — 源码位置；`fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 不适用（构造函数）
- **调用（向下）**: `deviant::deviant(loc, fmt, args...)`
- **被调用（向上）**: 需要显式指定源位置的场景
- **知识域**: 指定位置的安全异常构造

#### security::type_name() [protected]

- **功能说明**: 获取异常类型名称。返回固定值 `SECURITY`。
- **签名**: `[[nodiscard]] std::string_view type_name() const noexcept override`
- **参数**: 无
- **返回值**: `std::string_view` — 固定为 `"SECURITY"`
- **调用（向下）**: 无
- **被调用（向上）**: `deviant::dump()`
- **知识域**: 安全异常类型标识

---

## exception 概述

Exception 模块的详细设计文档。异常层次结构：`deviant`（抽象基类）-> `network` / `protocol` / `security`。每个子类提供 5 个构造函数重载（错误码、错误码+描述、字符串、格式化、格式化+位置），并实现 `type_name()` 虚函数。

## 相关页面

- [[fault/code|fault::code]] — 错误码枚举，异常构造使用
- [[fault/compatible|compatible]] — error_code 适配，异常内部使用
- [[agent/config|config]] — 配置加载阶段抛出异常
- [[loader/load|load]] — 配置加载器抛出 security/network 异常
- [[agent/worker/tls|tls]] — TLS 配置阶段抛出 protocol 异常
