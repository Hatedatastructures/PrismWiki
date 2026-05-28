---
tags: [trace, spdlog]
layer: core
source: src/prism/trace/spdlog.cpp
title: Trace Spdlog
module: trace
updated: 2026-05-27
---

# Trace Spdlog

spdlog 日志系统的集成实现。核心设计：双层指针 + 异步队列 + overrun_oldest。

## 核心机制

### 双层指针策略

| 层级 | 类型 | 用途 |
|------|------|------|
| `shared_system_logger` | `shared_ptr<logger>` | 管理 logger 生命周期 |
| `atomic_logger_ptr` | `atomic<logger*>` | 热路径无锁读取 |

`recorder()` 热路径先读原子指针（acquire load），避免 `shared_ptr` 引用计数的原子操作。只有 init/shutdown 时才访问 `shared_ptr`。

### 异步策略

```cpp
spdlog::async_logger(name, sinks, thread_pool, overrun_oldest)
```

`overrun_oldest`：队列满时丢弃最旧日志而非阻塞。日志不应阻塞网络 I/O 协程。

### Sink 组合

| Sink | 条件 | 说明 |
|------|------|------|
| `rotating_file_sink_mt` | `enable_file` | 按大小轮转，最多 `max_files` 个 |
| `stdout_color_sink_mt` | `enable_console` | 彩色控制台输出 |

## 核心函数

| 函数 | 说明 |
|------|------|
| `init(cfg)` | 创建目录 → 初始化线程池 → 创建 sinks → 配置 async logger |
| `shutdown()` | 清空原子指针 → flush → 释放 logger → `spdlog::shutdown()` |
| `recorder()` | 热路径：原子指针 load → `spdlog::default_logger()` |

## 约束

### init/shutdown 必须单线程调用

**类型**: 线程安全

**规则**: `init()` 和 `shutdown()` 使用 `shared_mutex` 保护，但设计上仅在 main 线程启动/关闭时调用。

**违反后果**: 并发调用 init/shutdown 可能导致 logger 指针状态不一致。

### recorder() 可多线程调用

**类型**: 线程安全

**规则**: `recorder()` 仅做 `atomic load`，无锁，可在任何线程安全调用。spdlog 的 `logger::log()` 内部线程安全。

## 日志级别映射

| 配置字符串 | spdlog 级别 |
|-----------|------------|
| `trace` | `trace` |
| `debug` | `debug` |
| `info` | `info` |
| `warn`/`warning` | `warn` |
| `error`/`err` | `err` |
| `critical`/`fatal` | `critical` |
| `off` | `off` |

大小写不敏感，默认 `info`。

## 引用关系

### 被引用

- [[core/trace/overview|overview]]：模块入口
- [[core/trace/config|config]]：配置参数
