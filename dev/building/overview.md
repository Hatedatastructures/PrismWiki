---
layer: dev
title: Prism 构建系统概述
created: 2026-05-17
updated: 2026-05-17
type: concept
tags: [build, cmake, overview, development]
related:
  - "[[dev/building/cmake]]"
  - "[[dev/building/dependencies]]"
  - "[[dev/building/commands]]"
  - "[[dev/building/options]]"
  - "[[dev/building/configuration|configuration]]"
sources:
  - I:/code/Prism/CLAUDE.md
confidence: high
layer: dev
---

# Prism 构建系统概述

Prism 使用 CMake 构建系统，采用现代 C++23 标准，支持跨平台编译。所有依赖通过 FetchContent 自动管理，简化了开发环境配置。

## 构建系统特点

### 现代化工具链

- **CMake 3.23+** — 现代 CMake 实践，支持预设和 FetchContent
- **C++23 编译器** — GCC 13+ 或 MinGW（Windows）
- **静态链接** — Windows 上使用 MinGW 静态链接，生成独立可执行文件

### 依赖管理策略

Prism 采用 **FetchContent 自动拉取** 策略：

- 所有依赖在配置阶段自动下载
- 无需手动安装本地库
- 确保依赖版本一致性
- 支持离线构建（已下载依赖）

### 构建产物

| 产物 | 描述 | 位置 |
|------|------|------|
| `Prism.exe` | 主程序可执行文件 | `build_release/src/` |
| `prism_static_library` | 核心静态库 | 链接至测试和基准 |
| 测试可执行文件 | 42 个独立测试 | `build_release/tests/` |
| 基准测试 | 12 个性能基准 | `build_release/benchmarks/` |
| 压力测试 | 4 个压力测试工具 | `build_release/stresses/` |

## 构建流程

```
配置 (cmake -B) → 编译 (cmake --build) → 测试 (ctest) → 部署
```

### 典型开发周期

```bash
# 1. 配置 Debug 构建（开发调试）
cmake -B build_debug -DCMAKE_BUILD_TYPE=Debug

# 2. 编译
cmake --build build_debug --config Debug

# 3. 运行相关测试
build_debug/tests/Session.exe
build_debug/tests/Socks5.exe

# 4. 配置 Release 构建（性能测试/发布）
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release

# 5. 运行完整测试套件
ctest --test-dir build_release --output-on-failure

# 6. 运行基准测试
build_release/benchmarks/SessionBench.exe
```

## 构建目录结构

```
Prism/
├── CMakeLists.txt          # 根配置
├── src/
│   └── CMakeLists.txt      # 主程序构建
├── tests/
│   └── CMakeLists.txt      # 测试构建
├── benchmarks/
│   └── CMakeLists.txt      # 基准测试构建
├── stresses/
│   └── CMakeLists.txt      # 压力测试构建
├── build_debug/            # Debug 构建目录
│   ├── src/Prism.exe
│   └── tests/*.exe
└── build_release/         # Release 构建目录
    ├── src/Prism.exe
    ├── tests/*.exe
    ├── benchmarks/*.exe
    └── stresses/*.exe
```

## 相关页面

- [[dev/building/cmake|CMake 构建结构]] — CMake 配置详解
- [[dev/building/dependencies|依赖项]] — 外部依赖说明
- [[dev/building/commands|构建命令]] — 常用构建命令
- [[dev/building/options|构建选项]] — 可配置构建选项