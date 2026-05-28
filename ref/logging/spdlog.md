---
title: spdlog 日志库
created: 2026-05-17
updated: 2026-05-27
layer: ref
tags: [ref, logging, spdlog, cpp]
---

# spdlog 日志库

> spdlog 是高性能 C++ 日志库，Prism 使用其作为 trace 模块的后端。本文介绍 spdlog 的架构、异步日志、Sink 类型、格式化模式以及与 Prism trace 模块的集成方式。

---

## 架构概览

### 核心组件

spdlog 的架构围绕三个核心概念：

```
Logger（日志器）
  │
  ├── Formatter（格式化器）── 格式化日志消息
  │
  └── Sink（输出目标）── 写入日志消息
        │
        ├── Sink A（如文件）
        ├── Sink B（如控制台）
        └── Sink C（如网络）
```

| 组件 | 职责 | 可定制性 |
|------|------|----------|
| Logger | 接收日志调用，分发到 Sink | 可注册多个 Logger |
| Formatter | 格式化日志消息文本 | 自定义 pattern |
| Sink | 输出到具体目标 | 多种内置 Sink |

### 注册表（Registry）

spdlog 维护一个全局 Logger 注册表：

```cpp
// 获取/创建 Logger
auto logger = spdlog::get("my_logger");
auto logger = spdlog::stdout_color_mt("console");

// 设置默认 Logger
spdlog::set_default_logger(logger);

// 全局日志级别
spdlog::set_level(spdlog::level::debug);
```

---

## 日志级别

| 级别 | 枚举值 | 用途 |
|------|--------|------|
| trace | `level::trace` | 最详细跟踪信息 |
| debug | `level::debug` | 调试信息 |
| info | `level::info` | 一般信息 |
| warn | `level::warn` | 警告 |
| error | `level::err` | 错误 |
| critical | `level::critical` | 严重错误 |
| off | `level::off` | 关闭日志 |

```cpp
// 按级别输出
logger->trace("trace message");
logger->debug("debug message");
logger->info("info message");
logger->warn("warn message");
logger->error("error message");
logger->critical("critical message");

// 条件输出（性能友好，级别不匹配时零开销）
SPDLOG_LOGGER_TRACE(logger, "only in trace mode: {}", value);
SPDLOG_LOGGER_DEBUG(logger, "debug: {}", value);
```

---

## Sink 类型

### 内置 Sink

| Sink | 头文件 | 说明 |
|------|--------|------|
| `stdout_color_sink` | `spdlog/sinks/stdout_color_sinks.h` | 彩色控制台输出 |
| `stdout_sink` | `spdlog/sinks/stdout_sinks.h` | 标准输出 |
| `basic_file_sink` | `spdlog/sinks/basic_file_sink.h` | 单文件输出 |
| `rotating_file_sink` | `spdlog/sinks/rotating_file_sink.h` | 滚动文件（按大小） |
| `daily_file_sink` | `spdlog/sinks/daily_file_sink.h` | 按天滚动 |
| `null_sink` | `spdlog/sinks/null_sink.h` | 丢弃所有日志 |
| `ostream_sink` | `spdlog/sinks/ostream_sink.h` | 输出到 std::ostream |

### 文件滚动 Sink

按大小滚动是最常用的文件 Sink：

```cpp
#include <spdlog/sinks/rotating_file_sink.h>

// 参数：文件名、单文件最大大小、最大文件数
auto sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>(
    "logs/prism.log",   // 文件名
    1024 * 1024 * 50,   // 50MB 单文件上限
    10                  // 保留 10 个文件
);

auto logger = std::make_shared<spdlog::logger>("prism", sink);
spdlog::set_default_logger(logger);
```

按天滚动：

```cpp
#include <spdlog/sinks/daily_file_sink.h>

auto sink = std::make_shared<spdlog::sinks::daily_file_sink_mt>(
    "logs/prism.log",  // 文件名
    0,                  // 轮换小时（0=午夜）
    30                  // 保留天数
);
```

### 多 Sink 组合

一个 Logger 可以绑定多个 Sink：

```cpp
auto console_sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
auto file_sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>(
    "logs/prism.log", 1024 * 1024 * 50, 10);

// 控制台只显示 info 及以上，文件记录全部
console_sink->set_level(spdlog::level::info);
file_sink->set_level(spdlog::level::trace);

std::vector<spdlog::sink_ptr> sinks{console_sink, file_sink};
auto logger = std::make_shared<spdlog::logger>("prism", sinks.begin(), sinks.end());
logger->set_level(spdlog::level::trace);
spdlog::set_default_logger(logger);
```

---

## 异步日志

### 为什么需要异步日志

同步日志的问题：

| 问题 | 说明 |
|------|------|
| 阻塞 I/O | 磁盘写入阻塞调用线程 |
| 锁竞争 | 多线程写日志争用 Sink 锁 |
| 延迟抖动 | I/O 延迟传导到业务逻辑 |

异步日志解决以上问题：日志调用仅将消息入队，由后台线程负责写入。

### 线程池

```cpp
#include <spdlog/async.h>

// 创建线程池（队列大小、工作线程数）
auto pool = spdlog::thread_pool_create(8192, 1);

// 使用线程池创建异步 Logger
auto logger = std::make_shared<spdlog::async_logger>(
    "async_prism",
    sinks.begin(), sinks.end(),
    pool
);
spdlog::set_default_logger(logger);
```

### 队列溢出策略

| 策略 | 枚举值 | 行为 |
|------|--------|------|
| 阻塞 | `block` | 入队线程阻塞等待（默认） |
| 丢弃新 | `overrun_oldest` | 丢弃最旧的消息腾出空间 |

```cpp
// 全局溢出策略
spdlog::set_async_mode(8192, spdlog::async_overflow_policy::overrun_oldest);
```

### 异步日志性能

| 指标 | 同步 | 异步 |
|------|------|------|
| 单次调用耗时 | ~1-10us（含 I/O） | ~50-200ns（仅入队） |
| 线程安全 | 需要锁 | 队列无锁操作 |
| 适用场景 | 单线程/低频 | 多线程/高频 |

---

## 格式化模式

### Pattern 格式

```cpp
logger->set_pattern("[%Y-%m-%d %H:%M:%S.%e] [%^%l%$] [thread %t] %v");
```

**常用标志**：

| 标志 | 含义 | 示例输出 |
|------|------|----------|
| `%Y` | 四位年份 | 2026 |
| `%m` | 月份 | 05 |
| `%d` | 日期 | 27 |
| `%H:%M:%S` | 时:分:秒 | 14:30:05 |
| `%e` | 毫秒 | 123 |
| `%f` | 微秒 | 123456 |
| `%l` | 日志级别 | info |
| `%L` | 级别缩写 | I |
| `%t` | 线程 ID | 12345 |
| `%n` | Logger 名称 | prism |
| `%v` | 日志消息 | 实际文本 |
| `%^...%$` | 颜色范围 | （终端着色） |

### Prism 推荐格式

```cpp
// 开发模式：详细信息
spdlog::set_pattern("[%Y-%m-%d %H:%M:%S.%e] [%^%l%$] [%s:%#] [%t] %v");

// 生产模式：简洁格式
spdlog::set_pattern("[%Y-%m-%d %H:%M:%S] [%^%l%$] %v");
```

---

## Prism trace 模块集成

### 集成架构

Prism 的 `trace` 模块是对 spdlog 的封装，提供模块化的日志接口：

```
trace::init()
  │
  ├── 创建异步线程池
  ├── 配置 Sink（控制台 + 滚动文件）
  ├── 设置格式化模式
  └── 注册模块级 Logger
        │
        ├── trace::session  — 会话日志
        ├── trace::tunnel   — 隧道日志
        ├── trace::dns      — DNS 解析日志
        ├── trace::stealth  — 伪装模块日志
        └── trace::pool     — 连接池日志
```

### 初始化

`trace::init()` 在 `main.cpp` 中调用，位于启动流程的早期：

```cpp
// main.cpp 启动顺序
memory::system::enable_global_pooling();
stealth::register_schemes();
loader::load(config);
trace::init(config.log);    // ← 初始化日志
loader::build_account_directory();
```

### 模块级日志

trace 模块按功能域划分日志通道，便于过滤和排查：

```cpp
namespace psm::trace {

// 模块日志器
auto session() -> spdlog::logger&;
auto tunnel() -> spdlog::logger&;
auto dns() -> spdlog::logger&;
auto stealth() -> spdlog::logger&;
auto pool() -> spdlog::logger&;

} // namespace psm::trace
```

使用示例：

```cpp
// 会话模块日志
trace::session().info("new connection from {}:{}", addr, port);
trace::session().debug("protocol detected: {}", protocol_name);

// 隧道模块日志
trace::tunnel().info("tunnel established: {} <-> {}:{}", src, target_host, target_port);

// DNS 模块日志
trace::dns().debug("cache hit: {} -> {}", domain, ip);
trace::dns().warn("upstream timeout: {}", domain);
```

### 日志文件滚动

Prism 使用 `rotating_file_sink` 管理日志文件：

- 单文件上限：50MB
- 保留文件数：10
- 总日志量上限：~500MB
- 文件命名：`prism.log`, `prism.1.log`, `prism.2.log`, ...

---

## 最佳实践

### 1. 避免在热路径使用格式化字符串

```cpp
// 错误：即使日志级别不匹配也会格式化字符串
trace::session().info("data: " + expensive_to_string(data));

// 正确：级别不匹配时跳过格式化
trace::session().info("data: {}", data);  // fmt 库惰性求值
```

### 2. 使用条件日志宏减少开销

```cpp
// SPDLOG_LOGGER_TRACE 在非 trace 级别时零开销
SPDLOG_LOGGER_TRACE(&trace::session(), "packet hex: {}", to_hex(buffer));
```

### 3. 不要在协程中同步刷新

```cpp
// 错误：阻塞协程
spdlog::default_logger()->flush();  // 可能阻塞数毫秒

// 正确：依赖异步 Logger 自动刷新，或设置定时刷新
spdlog::flush_every(std::chrono::seconds(5));
```

### 4. 合理设置日志级别

| 环境 | 推荐级别 | 说明 |
|------|----------|------|
| 开发 | trace/debug | 完整信息 |
| 测试 | debug/info | 关键调试信息 |
| 生产 | info/warn | 仅重要事件 |
| 排障 | debug | 临时开启详细日志 |

---

## 相关参考

- spdlog 官方仓库: https://github.com/gabime/spdlog
- spdlog Wiki: https://github.com/gabime/spdlog/wiki
- fmt 格式语法: https://fmt.dev/latest/syntax.html