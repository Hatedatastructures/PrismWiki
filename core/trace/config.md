---
tags: [trace, config]
layer: core
source: include/prism/trace/config.hpp
title: Trace Config
module: trace
updated: 2026-05-27
---

# Trace Config

日志系统配置参数。

## 配置字段

### 文件输出

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `file_name` | `"prism.log"` | 日志文件名 |
| `path_name` | `"logs"` | 存储路径 |
| `max_size` | 64MB | 单文件最大大小 |
| `max_files` | 8 | 最大保留文件数 |
| `enable_file` | `true` | 是否启用文件输出 |

### 异步配置

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `queue_size` | 8192 | 异步队列大小 |
| `thread_count` | 1 | 后台刷盘线程数 |

队列满时采用 `overrun_oldest` 策略，丢弃最旧日志。生产环境队列大小应远大于峰值日志速率。

### 控制台与格式

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `enable_console` | `true` | 控制台输出 |
| `log_level` | `"info"` | 日志级别 |
| `pattern` | `[%Y-%m-%d %H:%M:%S.%e][%l] %v` | 输出格式 |
| `trace_name` | `"prism"` | 日志器名称 |

## 设计决策

### 为什么字符串字段用 memory::string？

`trace::config` 在启动阶段构造，字符串字段（file_name、path_name 等）生命周期贯穿进程。`memory::string` 使用 PMR 分配，与项目内存策略一致。

**后果**: 构造 config 时需指定内存资源，默认使用 `memory::system::global_pool()`。

## 引用关系

### 被引用

- [[core/trace/spdlog|spdlog]]：`init()` 消费 config 创建 logger
- [[core/trace/overview|overview]]：模块总览
