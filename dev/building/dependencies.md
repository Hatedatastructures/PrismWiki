---
title: 构建依赖项
created: 2026-05-17
updated: 2026-05-17
type: reference
tags: [dependencies, cmake, build, external-libraries]
related:
  - "[[dev/building/overview]]"
  - "[[dev/building/cmake]]"
  - "[[dev/building/commands]]"
  - "[[crypto/overview]]"
sources:
  - I:/code/Prism/CLAUDE.md
  - I:/code/Prism/CMakeLists.txt
confidence: high
---

# 构建依赖项

Prism 采用 FetchContent 自动依赖管理，无需手动安装外部库。本文档列出所有构建依赖及其用途。

## 编译器要求

| 要求 | 说明 |
|------|------|
| C++23 编译器 | GCC 13+ 或 MinGW（Windows） |
| CMake 3.23+ | 现代 CMake 特性支持 |
| NASM 汇编器 | BoringSSL 汇编优化（自动下载） |

### Windows 特定

Windows 平台使用 MinGW 静态链接：

- `-static` — 全静态链接
- `-static-libgcc` — 静态链接 GCC 运行时
- `-static-libstdc++` — 静态链接 C++ 标准库

## 外部依赖

### Boost.Asio

```cmake
FetchContent_Declare(
  boost_asio
  GIT_REPOSITORY https://github.com/boostorg/asio.git
  GIT_TAG        master
)
```

**用途**：
- 协程支持（`net::awaitable<T>`）
- 异步 I/O 操作
- 网络编程抽象层

**特点**：
- Header-only 库
- 无需编译 Boost
- 提供 `co_await` 异步模型

### BoringSSL

```cmake
FetchContent_Declare(
  boringssl
  GIT_REPOSITORY https://github.com/google/boringssl.git
  GIT_TAG        <commit-hash>
)
```

**用途**：
- TLS 1.2/1.3 实现
- 加密原语（AEAD、X25519 等）
- 证书验证

**特点**：
- OpenSSL API 兼容
- Google 维护的 TLS 库
- 自动应用 MinGW 兼容性 patch

### spdlog

```cmake
FetchContent_Declare(
  spdlog
  GIT_REPOSITORY https://github.com/gabime/spdlog.git
  GIT_TAG        v1.x
)
```

**用途**：
- 高性能日志记录
- 异步日志输出
- 日志轮转和级别控制

**特点**：
- Header-only 或编译模式
- 支持多 sink 输出
- 低延迟异步日志

### glaze

```cmake
FetchContent_Declare(
  glaze
  GIT_REPOSITORY https://github.com/stephenberry/glaze.git
  GIT_TAG        <version>
)
```

**用途**：
- JSON 序列化/反序列化
- 配置文件解析
- 高性能 JSON 处理

**特点**：
- 纯头文件或编译模式
- 编译期反射
- 零拷贝设计

### BLAKE3

```cmake
FetchContent_Declare(
  blake3
  GIT_REPOSITORY https://github.com/BLAKE3-team/BLAKE3.git
  GIT_TAG        v1.8.1
)
```

**用途**：
- SS2022 密钥派生
- 高性能哈希函数
- 密码学安全哈希

**特点**：
- 极高性能（SIMD 优化）
- 固定版本 v1.8.1
- 用于 Shadowsocks 2022 协议

## 测试依赖

### Google Benchmark

```cmake
FetchContent_Declare(
  benchmark
  GIT_REPOSITORY https://github.com/google/benchmark.git
  GIT_TAG        v1.8.0
)
set(BENCHMARK_ENABLE_TESTING OFF)
```

**用途**：
- 性能基准测试框架
- 微基准测试
- 性能回归检测

**特点**：
- 仅在启用 `PRISM_ENABLE_BENCHMARK=ON` 时拉取
- 自动禁用自身测试

## 系统库依赖

### Windows 平台

```cmake
set(SYSTEM_LIBRARIES
  ws2_32    # Windows Sockets API
  mswsock   # Microsoft Windows Sockets 扩展
  crypt32   # Windows Crypto API
)
```

| 库 | 用途 |
|----|------|
| `ws2_32` | 基础 socket 操作 |
| `mswsock` | 高性能 socket 扩展 |
| `crypt32` | 证书存储访问 |

### Linux 平台

Linux 平台通常无需额外系统库，标准 C 库和 POSIX API 已足够。

## 依赖关系图

```
Prism
├── boost_asio (协程/I/O)
├── boringssl (TLS/加密)
│   └── NASM (汇编优化, 自动下载)
├── spdlog (日志)
├── glaze (JSON)
├── blake3 (哈希)
└── [可选] google benchmark (性能测试)
```

## FetchContent 工作流程

```
cmake -B build
    │
    ▼
FetchContent 声明所有依赖
    │
    ▼
检查 _deps/ 目录
    │
    ├── 已存在 → 跳过下载
    │
    └── 不存在 → git clone 到 _deps/
                    │
                    ▼
                 配置依赖构建
                    │
                    ▼
                 添加到目标链接
```

## 离线构建

依赖下载后存储在 `_deps/` 目录：

```bash
# 首次构建（联网）
cmake -B build_release -DCMAKE_BUILD_TYPE=Release

# 后续构建（可离线）
cmake --build build_release --config Release
```

## 相关页面

- [[dev/building/overview|构建系统概述]] — 构建流程概览
- [[dev/building/cmake|CMake 构建结构]] — CMake 配置详解
- [[dev/building/commands|构建命令]] — 常用构建命令
- [[crypto/overview|加密模块]] — 加密相关依赖用途