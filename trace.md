---
title: "Trace 模块"
module: "trace"
type: module
tags: [trace, spdlog, 日志, 异步, config]
created: 2026-05-15
updated: 2026-05-15
related:
  - memory/container
  - loader/load
  - agent/config
  - channel/connection/pool
  - resolve/router
---

# Trace 模块

> 模块: [[trace|trace]]

## 概述

Trace 模块负责异步日志记录，基于 spdlog 库实现。提供高性能的异步日志输出，支持多后端（控制台、文件轮转）。所有接口内部捕获异常，确保日志失败不影响业务逻辑。日志器使用 `atomic<logger*>` 实现无锁读取，仅在 `init/shutdown` 时加写锁。

---

## config.hpp — 日志配置

> 源码: `include/prism/trace/config.hpp`
> 模块: [[trace|trace]]

### 概述

日志配置结构。定义 spdlog 日志系统的配置参数，支持异步日志记录、日志轮转和多目标输出。所有字符串字段使用 `memory::string` 类型，配合 PMR 分配器。

### 命名空间

`psm::trace`

### 结构体: config

- **功能说明**: 日志配置参数。配置 spdlog 日志系统的各项参数，包括文件输出、控制台输出、日志轮转和异步队列。
- **签名**: `struct config`

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `file_name` | `memory::string` | `"prism.log"` | 日志文件名 |
| `path_name` | `memory::string` | `"logs"` | 日志文件存储路径 |
| `max_size` | `std::uint64_t` | `67108864` (64MB) | 单个日志文件最大大小 |
| `max_files` | `std::uint32_t` | `8` | 最大轮转文件数量 |
| `queue_size` | `std::uint32_t` | `8192` | 异步队列大小 |
| `thread_count` | `std::uint32_t` | `1` | 后台刷盘线程数 |
| `enable_console` | `bool` | `true` | 是否启用控制台输出 |
| `enable_file` | `bool` | `true` | 是否启用文件输出 |
| `log_level` | `memory::string` | `"info"` | 日志级别（trace/debug/info/warn/error/critical/off） |
| `pattern` | `memory::string` | `"[%Y-%m-%d %H:%M:%S.%e][%l] %v"` | 日志格式（spdlog 格式语法） |
| `trace_name` | `memory::string` | `"prism"` | 日志追踪名称，用于 logger 命名和 spdlog::drop |

- **返回值**: 不适用（结构体定义）
- **调用（向下）**: 无
- **被调用（向上）**: `trace::init()`、`loader::load()` 返回的 `config.trace` 字段
- **知识域**: 日志系统配置

---

## spdlog.hpp — spdlog 异步日志封装

> 源码: `include/prism/trace/spdlog.hpp`
> 实现: `src/prism/trace/spdlog.cpp`
> 模块: [[trace|trace]]

### 概述

spdlog 异步日志封装。初始化 spdlog 异步 logger，配置日志格式、级别和输出目标。日志模板函数（debug/info/warn/error/fatal）在未初始化时安全跳过，不产生任何输出。

### 命名空间

`psm::trace`

### 函数: init()

- **功能说明**: 初始化全局日志器。根据配置创建异步日志器，支持文件和控制台双输出。如果已存在日志器，会先销毁旧的再创建新的。创建线程池、构建 sink 列表、设置日志级别和格式。
- **签名**: `void init(const config &cfg)`
- **参数**: `cfg` — 日志配置对象
- **返回值**: 无
- **调用（向下）**: `spdlog::init_thread_pool()`、`spdlog::sinks::rotating_file_sink_mt` 构造、`spdlog::sinks::stdout_color_sink_mt` 构造、`spdlog::async_logger` 构造、`spdlog::set_default_logger()`、`spdlog::set_level()`、`parse_spdlog_level()`、`build_log_path()`
- **被调用（向上）**: `main.cpp` 启动流程（`trace::init(full_config.trace)`）
- **知识域**: spdlog 异步日志初始化

### 函数: shutdown()

- **功能说明**: 关闭日志器并释放线程池资源。将缓冲区日志刷盘后释放资源，调用后所有日志接口不再产生输出，直到再次调用 `init()`。使用写锁保护 `shared_system_logger` 的移动。
- **签名**: `void shutdown()`
- **参数**: 无
- **返回值**: 无
- **调用（向下）**: `spdlog::logger::flush()`、`spdlog::drop()`、`spdlog::shutdown()`
- **被调用（向上）**: 程序退出前清理
- **知识域**: spdlog 资源释放

### 函数: recorder()

- **功能说明**: 获取当前日志器。使用 `atomic<logger*>` 无锁读取，减少热路径锁开销。日志器指针仅在 `init/shutdown` 时变更。
- **签名**: `[[nodiscard]] std::shared_ptr<spdlog::logger> recorder() noexcept`
- **参数**: 无
- **返回值**: `std::shared_ptr<spdlog::logger>` — 全局日志器的 shared_ptr，未初始化则为空
- **调用（向下）**: `atomic_logger_ptr.load()`、`spdlog::default_logger()`
- **被调用（向上）**: `debug()`、`info()`、`warn()`、`error()`、`fatal()`
- **知识域**: 无锁日志器访问

### 函数模板: debug()

- **功能说明**: 记录调试日志。如果日志器未初始化则安全跳过。
- **签名**: `template <typename... Args> void debug(std::string_view fmt, Args &&...args)`
- **参数**: `fmt` — 格式化字符串（支持 fmt 库语法）；`args` — 格式化参数
- **返回值**: 无
- **调用（向下）**: `recorder()`、`spdlog::logger::debug()`
- **被调用（向上）**: 连接池（Pool）、Happy Eyeballs（Racer）、DNS 解析（Resolve）、出站代理（Outbound.Direct）等模块的调试日志
- **知识域**: 调试级别日志记录

### 函数模板: info()

- **功能说明**: 记录信息日志。如果日志器未初始化则安全跳过。
- **签名**: `template <typename... Args> void info(std::string_view fmt, Args &&...args)`
- **参数**: `fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 无
- **调用（向下）**: `recorder()`、`spdlog::logger::info()`
- **被调用（向上）**: 连接建立成功、Happy Eyeballs 胜出端点、出站代理连接成功等关键路径日志
- **知识域**: 信息级别日志记录

### 函数模板: warn()

- **功能说明**: 记录警告日志。如果日志器未初始化则安全跳过。
- **签名**: `template <typename... Args> void warn(std::string_view fmt, Args &&...args)`
- **参数**: `fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 无
- **调用（向下）**: `recorder()`、`spdlog::logger::warn()`
- **被调用（向上）**: 连接池超时、TCP 选项设置失败、DNS 解析失败、路由失败、识别失败等警告日志
- **知识域**: 警告级别日志记录

### 函数模板: error()

- **功能说明**: 记录错误日志。如果日志器未初始化则安全跳过。
- **签名**: `template <typename... Args> void error(std::string_view fmt, Args &&...args)`
- **参数**: `fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 无
- **调用（向下）**: `recorder()`、`spdlog::logger::error()`
- **被调用（向上）**: `main.cpp` 异常捕获、X25519 密钥错误、HKDF 错误、AEAD 错误、连接池清理错误、Happy Eyeballs 错误等严重错误日志
- **知识域**: 错误级别日志记录

### 函数模板: fatal()

- **功能说明**: 记录致命错误日志。映射到 spdlog 的 `critical` 级别。致命错误通常意味着程序即将终止，但此函数不会自动终止程序。
- **签名**: `template <typename... Args> void fatal(std::string_view fmt, Args &&...args)`
- **参数**: `fmt` — 格式化字符串；`args` — 格式化参数
- **返回值**: 无
- **调用（向下）**: `recorder()`、`spdlog::logger::critical()`
- **被调用（向上）**: 致命错误场景
- **知识域**: 致命级别日志记录

---

## trace 概述

Trace 模块的详细设计文档。基于 spdlog 的异步日志系统，使用 `atomic<logger*>` 实现热路径无锁读取。支持文件轮转（默认 64MB x 8 文件）和控制台彩色输出。日志级别支持 trace/debug/info/warn/error/critical/off 七级。

## 相关页面

- [[memory/container|container]] — PMR 容器，config 中的 string 字段使用
- [[loader/load|load]] — 配置加载器，解析 JSON 后调用 `trace::error()` 记录错误
- [[agent/config|config]] — Agent 配置，包含 `trace::config` 字段
- [[channel/connection/pool|pool]] — 连接池，使用 trace 记录连接生命周期
- [[resolve/router|router]] — 路由器，使用 trace 记录 DNS 解析结果
