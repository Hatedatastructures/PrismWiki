---
title: Core 基础设施
created: 2026-05-17
updated: 2026-05-28
layer: core
source: "include/prism/memory/, include/prism/fault/, include/prism/exception/, include/prism/trace/, include/prism/transformer/, include/prism/loader/"
tags: [infrastructure, memory, fault, exception, trace, transformer, loader]
---

# Core 基础设施

Core 基础设施模块提供系统的底层支撑，包括内存管理、错误处理、异常系统、日志记录、序列化和配置加载。

## 模块清单

| 模块 | 说明 | 页面 |
|------|------|------|
| **Memory** | PMR内存管理 | [[core/memory/overview]] |
| **Fault** | 错误码系统 | [[core/fault/overview]] |
| **Exception** | 异常系统 | [[core/exception/overview]] |
| **Trace** | 日志系统 | [[core/trace/overview]] |
| **Transformer** | 序列化 | [[core/transformer/overview]] |
| **Loader** | 配置加载 | [[core/loader/overview]] |

## 核心设计原则

### 热路径无异常

网络I/O、协议解析等高频路径使用错误码返回值，不抛异常。`fault::code` 枚举 + `fault::failed()` 检查。禁止在热路径中 `throw exception::network(...)`。

### 热路径无分配

使用线程局部内存池避免动态分配。PMR 容器（`memory::vector`、`memory::string`）配合 `memory::system::hot_path_pool()` 实现零堆分配。

### 启动阶段异常

配置加载、资源初始化等启动阶段使用异常：`loader::load(path)` 失败时抛出 `exception::security`，由顶层 catch 处理。

## 启动流程

`loader::load` 加载配置 → `trace::init` 初始化日志 → `build_account_directory` 构建账户目录 → 启动服务。任一步骤失败均抛出异常终止启动。

## 模块依赖关系

| 依赖方 | 提供方 | 依赖内容 |
|--------|--------|----------|
| Exception | Fault | `std::error_code` 基础 |
| Trace | Memory | PMR字符串配置 |
| Loader | Transformer | JSON反序列化 |
| Loader | Exception | 启动阶段异常 |
| Loader | Trace | 错误日志 |

## 相关页面

- [[core/memory/overview]] - Memory模块
- [[core/fault/overview]] - Fault模块
- [[core/exception/overview]] - Exception模块
- [[core/trace/overview]] - Trace模块
- [[core/transformer/overview]] - Transformer模块
- [[core/loader/overview]] - Loader模块
