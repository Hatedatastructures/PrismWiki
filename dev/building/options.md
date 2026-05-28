---
layer: dev
title: 构建选项
created: 2026-05-17
updated: 2026-05-17
type: reference
tags: [cmake, options, configuration, build]
related:
  - "[[dev/building/overview]]"
  - "[[dev/building/cmake]]"
  - "[[dev/building/commands]]"
  - "[[dev/testing/testing|testing]]"
  - "[[dev/performance/report|performance/benchmark]]"
sources:
  - I:/code/Prism/CLAUDE.md
confidence: high
layer: dev
---

# 构建选项

本文档列出 Prism 项目可配置的 CMake 构建选项。

## 核心选项

### PRISM_ENABLE_BENCHMARK

```bash
-DRISM_ENABLE_BENCHMARK=ON   # 启用基准测试（默认）
-DRISM_ENABLE_BENCHMARK=OFF  # 禁用基准测试
```

**说明**：控制是否构建性能基准测试目标。

**默认值**：`ON`

**影响**：
- 启用时：拉取 Google Benchmark 依赖，构建 12 个基准测试
- 禁用时：跳过基准测试，减少编译时间

**基准测试列表**：
- CodecBench
- CryptoBench
- IOBench
- LatencyBench
- MemoryBench
- MuxBench
- MuxConnBench
- ProtocolBench
- RealityBench
- ResourceBench
- ScaleBench
- SessionBench

### PRISM_ENABLE_STRESS

```bash
-DRISM_ENABLE_STRESS=ON   # 启用压力测试（默认）
-DRISM_ENABLE_STRESS=OFF  # 禁用压力测试
```

**说明**：控制是否构建压力测试工具。

**默认值**：`ON`

**影响**：
- 启用时：构建 4 个压力测试工具
- 禁用时：跳过压力测试

**压力测试列表**：
- MemoryStress — 内存压力测试
- PoolContentionStress — 内存池竞争测试
- ArenaOverflowStress — 竞技场溢出测试
- MuxStress — 多路复用压力测试

## 标准构建类型

### CMAKE_BUILD_TYPE

```bash
-DCMAKE_BUILD_TYPE=Release   # 发布构建
-DCMAKE_BUILD_TYPE=Debug     # 调试构建
-DCMAKE_BUILD_TYPE=RelWithDebInfo  # 带调试信息的发布构建
-DCMAKE_BUILD_TYPE=MinSizeRel      # 最小尺寸发布构建
```

| 类型 | 优化 | 调试符号 | 用途 |
|------|------|----------|------|
| Release | `-O3` | 无 | 生产部署、性能测试 |
| Debug | `-O0` | 有 | 开发调试、断点调试 |
| RelWithDebInfo | `-O2` | 有 | 性能分析 |
| MinSizeRel | `-Os` | 无 | 尺寸优化部署 |

## 编译器选项

### C++ 标准

```bash
-DCMAKE_CXX_STANDARD=23  # C++23（必需）
```

**说明**：Prism 要求 C++23 标准，使用协程、Ranges 等特性。

### 优化选项（Release）

```
-O3                 # 最高优化级别
-ffunction-sections # 函数分段（优化链接）
-fdata-sections     # 数据分段（优化链接）
```

### 链接选项（Windows/MinGW）

```
-static            # 全静态链接
-static-libgcc     # 静态链接 GCC 运行时
-static-libstdc++  # 静态链接 C++ 标准库
```

### 大目标文件选项（Windows/MinGW）

```
-Wa,-mbig-obj      # 支持大目标文件
```

**用途**：避免测试编译时 "file too big" 错误。

## 配置示例

### 最小构建（仅核心）

```bash
cmake -B build_minimal \
  -DCMAKE_BUILD_TYPE=Release \
  -DPRISM_ENABLE_BENCHMARK=OFF \
  -DPRISM_ENABLE_STRESS=OFF

cmake --build build_minimal --config Release
```

**产物**：
- `Prism.exe` — 主程序
- `prism_static_library` — 静态库
- 测试可执行文件

**优点**：编译速度快，依赖少

### 完整构建（包含所有）

```bash
cmake -B build_full \
  -DCMAKE_BUILD_TYPE=Release \
  -DPRISM_ENABLE_BENCHMARK=ON \
  -DPRISM_ENABLE_STRESS=ON

cmake --build build_full --config Release
```

**产物**：
- 主程序
- 测试
- 基准测试
- 压力测试

### 调试构建

```bash
cmake -B build_debug \
  -DCMAKE_BUILD_TYPE=Debug \
  -DPRISM_ENABLE_BENCHMARK=OFF \
  -DPRISM_ENABLE_STRESS=OFF

cmake --build build_debug --config Debug
```

**特点**：
- 完整调试符号
- 禁用优化
- 快速迭代

### 性能分析构建

```bash
cmake -B build_profile \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DPRISM_ENABLE_BENCHMARK=ON \
  -DPRISM_ENABLE_STRESS=OFF

cmake --build build_profile --config RelWithDebInfo
```

**特点**：
- 带优化和调试符号
- 适合性能分析器
- 基准测试可用于对比

## 选项组合建议

| 场景 | BUILD_TYPE | BENCHMARK | STRESS |
|------|------------|-----------|--------|
| 生产部署 | Release | OFF | OFF |
| 开发调试 | Debug | OFF | OFF |
| 性能测试 | Release | ON | OFF |
| 压力测试 | Release | OFF | ON |
| CI/CD | Release | ON | ON |
| 性能分析 | RelWithDebInfo | ON | OFF |

## 相关页面

- [[dev/building/overview|构建系统概述]] — 构建流程概览
- [[dev/building/cmake|CMake 构建结构]] — CMake 配置详解
- [[dev/building/commands|构建命令]] — 常用构建命令
- [[dev/testing/testing|测试指南]] — 测试详情
- [[dev/performance/report|性能基准]] — 基准测试结果
- [[dev/testing/stress|压力测试]] — 压力测试详情