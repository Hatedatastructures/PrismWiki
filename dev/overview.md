---
title: Dev 层总览
created: 2026-05-17
layer: dev
tags: [dev, overview]
---

# Dev 层总览

Dev 层包含开发规范、排障方法、Bug 记录等开发相关内容。

## 职责

- 编码规范详解
- 测试体系说明
- 排障方法步骤
- 构建系统说明
- 性能优化指南
- 扩展开发指南
- Bug 记录归档

## 详细程度

与 [[core/overview|Core 层]] 相同，详细说明每条规范、每个步骤的原因和具体操作。

## 子目录

| 目录 | 职责 | 主要内容 |
|------|------|----------|
| [[coding/overview|coding]] | 编码规范 | 命名规范、协程约定、PMR 使用、注释规范、生命周期安全、错误处理 |
| [[testing/overview|testing]] | 测试体系 | 测试框架、命令、基准测试、压力测试 |
| [[debugging/overview|debugging]] | 排障方法 | 连接/协议/内存/性能/TLS 排障、日志分析 |
| [[building/overview|building]] | 构建系统 | CMake 结构、依赖管理、构建命令、构建选项 |
| [[dev/performance/overview|performance]] | 性能优化 | 调优方法、性能分析、性能报告 |
| [[extending/overview|extending]] | 扩展开发 | 新协议集成、新伪装方案、新模块开发 |
| [[bugs/template|bugs]] | Bug 记录 | Bug 报告模板、归档 |

## 开发路线图

详见 [[roadmap|开发路线图]]

## 快速导航

### 编码规范

- [[coding/naming|命名规范]] — snake_case 规则
- [[coding/coroutine|协程约定]] — 协程纯度要求（禁止表）
- [[coding/pmr|PMR 使用规范]] — 内存池使用方式
- [[coding/doxygen|注释规范]] — Doxygen 中文注释风格
- [[coding/lifecycle|生命周期安全]] — co_spawn 按值捕获
- [[coding/error|错误处理策略]] — 双轨策略：错误码 + 异常

### 测试

- [[testing/commands|测试命令]] — ctest 运行
- [[testing/benchmark|基准测试]] — 12 个基准测试
- [[testing/stress|压力测试]] — 4 个压力测试

### 构建

- [[building/commands|构建命令]] — cmake 配置与构建
- [[building/dependencies|依赖管理]] — FetchContent 自动获取
- [[building/options|构建选项]] — BENCHMARK/STRESS 开关

### 排障

- [[debugging/connection|连接问题]] — 连接断开、超时
- [[debugging/protocol|协议问题]] — 识别失败、处理错误
- [[debugging/memory|内存问题]] — PMR 泄漏、碎片
- [[debugging/performance|性能问题]] — 延迟高、吞吐低
- [[debugging/log-analysis|日志分析]] — 日志解读方法