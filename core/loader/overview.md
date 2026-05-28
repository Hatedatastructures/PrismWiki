---
tags: [loader, overview]
layer: core
module: loader
source: I:/code/Prism/include/prism/loader/load.hpp
title: Loader 模块
---

# Loader 模块

Loader 模块提供配置加载功能，将外部配置文件转换为内部配置结构。是外部 JSON 格式与内部类型之间的防腐层。

## 设计决策

### 为什么 loader 是防腐层？

外部配置文件格式（JSON 字段名、结构）和内部 `psm::config` 结构体是两个独立演化的东西。如果业务代码直接消费 JSON，格式变更会导致全项目改动。Loader 将 JSON 反序列化为中间结构，再转换为 `psm::config`，隔离格式变化。

**后果**: 配置格式变更只需修改 loader，不影响下游消费者。

### 为什么启动阶段用异常？

配置加载失败是致命错误，程序无法继续运行。异常自动展开调用栈，`main()` 的 `try/catch` 统一处理，不需要逐层检查错误码。

**后果**: Loader 抛出 `exception::security` 或 `exception::network`，调用方必须用 `try/catch`。

## 模块组成

| 组件 | 说明 | 源码 |
|------|------|------|
| [[core/loader/load]] | 配置加载器 | `prism/loader/load.hpp` |

## 核心接口

| 函数 | 说明 |
|------|------|
| `load(path)` | 加载配置文件，返回 `psm::config`。失败时抛异常 |
| `build_account_directory(auth)` | 从认证配置构建账户目录 |

## 约束

### load() 必须在 worker 创建前调用

**类型**: 调用顺序

**规则**: `loader::load()` 必须在创建 worker 线程池之前完成（`main.cpp` 启动流程保证）。

**违反后果**: worker 使用未初始化的配置，行为不可预测。

## 引用关系

### 依赖

- [[core/transformer/json|json]]：JSON 反序列化
- [[core/exception/overview|exception]]：启动阶段异常

### 被引用

- `main.cpp`：启动流程调用