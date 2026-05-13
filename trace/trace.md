---
title: Trace 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [trace, logging, spdlog, infrastructure, runtime]
related: [[memory], [agent/overview], [architecture]]
module: trace
category: infrastructure
layer: runtime
status: stable
dependencies:
  - spdlog
  - memory
dependents:
  - agent
  - channel
  - pipeline
  - outbound
---

# Trace 模块

## 1. 模块定位

Trace 模块是 Prism 的**日志基础设施**，封装 spdlog 库提供统一的日志记录接口。支持异步日志、日志轮转、多目标输出（控制台 + 文件）。

**核心价值**：提供高性能异步日志，所有接口内部捕获异常，确保日志失败不影响业务逻辑。

**边界**：Trace 模块负责日志基础设施，不包含业务逻辑。各模块通过 `trace::info()` 等模板函数输出日志。

## 2. 核心功能

### 2.1 日志配置

`trace::config` 结构体定义日志系统参数：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `file_name` | `memory::string` | `"prism.log"` | 日志文件名 |
| `path_name` | `memory::string` | `"logs"` | 日志存储路径 |
| `max_size` | `uint64_t` | 64MB | 单文件最大大小 |
| `max_files` | `uint32_t` | 8 | 最大文件数量 |
| `queue_size` | `uint32_t` | 8192 | 异步队列大小 |
| `thread_count` | `uint32_t` | 1 | 后台刷盘线程数 |
| `enable_console` | `bool` | true | 是否启用控制台输出 |
| `enable_file` | `bool` | true | 是否启用文件输出 |
| `log_level` | `memory::string` | `"info"` | 日志级别 |
| `pattern` | `memory::string` | `"[%Y-%m-%d %H:%M:%S.%e][%l] %v"` | 日志格式 |
| `trace_name` | `memory::string` | `"prism"` | 日志器名称 |

所有字符串字段使用 `memory::string`（PMR 分配器），与 [[memory]] 集成。

### 2.2 日志接口

提供五个级别的模板化日志函数：
- `trace::debug(fmt, args...)`
- `trace::info(fmt, args...)`
- `trace::warn(fmt, args...)`
- `trace::error(fmt, args...)`
- `trace::fatal(fmt, args...)` — 映射到 spdlog 的 `critical` 级别

所有函数内部检查 `recorder()` 是否为空，未初始化时安全忽略。

### 2.3 生命周期管理

- `init(cfg)` — 初始化异步日志器（支持重复调用，后一次覆盖前一次）
- `shutdown()` — 刷盘并释放线程池资源（程序退出前必须调用）
- `recorder()` — 获取底层 `spdlog::logger` 的 `shared_ptr`

## 3. 关键类和接口

### 3.1 `trace::config` — 日志配置（`config.hpp`）

```cpp
namespace psm::trace {
    struct config {
        memory::string file_name = "prism.log";
        memory::string path_name = "logs";
        std::uint64_t max_size = 64ULL * 1024ULL * 1024ULL;
        std::uint32_t max_files = 8U;
        std::uint32_t queue_size = 8192U;
        std::uint32_t thread_count = 1U;
        bool enable_console = true;
        bool enable_file = true;
        memory::string log_level = "info";
        memory::string pattern = "[%Y-%m-%d %H:%M:%S.%e][%l] %v";
        memory::string trace_name = "prism";
    };
}
```

### 3.2 生命周期管理（`spdlog.hpp`）

```cpp
namespace psm::trace {
    void init(const config &cfg);     // 初始化异步日志器（可重复调用）
    void shutdown();                   // 刷盘并释放线程池资源
    [[nodiscard]] std::shared_ptr<spdlog::logger> recorder() noexcept;
}
```

- `init()` 创建异步日志器，支持文件和控制台双 sink。已存在日志器时先销毁旧的再创建新的
- `shutdown()` 将缓冲区日志刷盘后释放资源
- `recorder()` 返回全局日志器的 `shared_ptr`，未初始化时返回空

### 3.3 日志模板函数（`spdlog.hpp`）

```cpp
template<typename... Args> void debug(std::string_view fmt, Args&&... args);
template<typename... Args> void info(std::string_view fmt, Args&&... args);
template<typename... Args> void warn(std::string_view fmt, Args&&... args);
template<typename... Args> void error(std::string_view fmt, Args&&... args);
template<typename... Args> void fatal(std::string_view fmt, Args&&... args);  // → spdlog::critical
```

实现示例：
```cpp
template<typename... Args>
void info(const std::string_view fmt, Args &&...args) {
    if (const auto rec = recorder()) {
        rec->info(spdlog::fmt_lib::runtime(fmt), std::forward<Args>(args)...);
    }
}
```

使用 `spdlog::fmt_lib::runtime()` 包装运行时格式字符串，避免编译期格式检查。

## 4. 使用模式

### 4.1 启动初始化
```cpp
trace::config cfg;
cfg.log_level = "debug";
cfg.enable_console = true;
trace::init(cfg);
```

### 4.2 日志输出
```cpp
trace::info("[Session] new connection from {}:{}", remote_ip, remote_port);
trace::warn("[Route] fallback to direct: {}", fault::describe(ec));
trace::error("[TLS] handshake failed: {}", error_msg);
```

### 4.3 程序退出
```cpp
trace::shutdown();  // 刷盘并释放资源
```

## 5. 文件清单

### 头文件（`include/prism/trace/`）

```
trace.hpp                # 聚合头文件
├── config.hpp           # 日志配置结构体（config）
└── spdlog.hpp           # 日志接口封装（init, shutdown, recorder, debug/info/warn/error/fatal）
```

### 源文件（`src/prism/trace/`）

```
└── spdlog.cpp           # init() 和 shutdown() 实现
```

## 6. 设计原理

- **异步日志**：使用 spdlog 的异步模式，日志写入由后台线程完成，不阻塞业务线程
- **异常安全**：所有日志模板函数内部检查 `recorder()` 是否为空，spdlog 内部也捕获异常
- **可重复初始化**：`init()` 支持重复调用，后一次覆盖前一次，允许运行时重新配置
- **日志轮转**：`max_size` 和 `max_files` 控制日志轮转，单文件达到上限后自动创建新文件
- **PMR 集成**：`config` 的字符串字段使用 `memory::string`

## 7. 外部依赖

| 依赖 | 用途 | 集成方式 |
|------|------|---------|
| spdlog | 异步日志库 | FetchContent |
| fmt | 格式化（spdlog 内置） | 随 spdlog |

## 相关页面

- [[memory]] — Memory 模块（config 使用 PMR 容器）
- [[agent]] — Agent 模块（启动时调用 trace::init）
- [[architecture]] — 架构设计
