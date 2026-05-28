---
layer: dev
title: CMake 构建结构
created: 2026-05-17
updated: 2026-05-17
type: reference
tags: [cmake, build, configuration, structure]
related:
  - "[[dev/building/overview]]"
  - "[[dev/building/dependencies]]"
  - "[[dev/building/commands]]"
  - "[[dev/building/options]]"
sources:
  - I:/code/Prism/CLAUDE.md
  - I:/code/Prism/CMakeLists.txt
confidence: high
layer: dev
---

# CMake 构建结构

Prism 采用分层 CMake 配置，根目录负责全局配置和依赖管理，子目录负责具体构建目标。

## 根目录 CMakeLists.txt

根配置负责以下核心任务：

### 标准和编译器设置

```cmake
# 设置 C++23 标准
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Release 优化选项
set(CMAKE_CXX_FLAGS_RELEASE "-O3 -ffunction-sections -fdata-sections")
```

### 依赖管理（FetchContent）

```cmake
include(FetchContent)

# 自动获取所有依赖
FetchContent_Declare(
  boost_asio
  GIT_REPOSITORY https://github.com/boostorg/asio.git
  GIT_TAG        master
)

FetchContent_Declare(
  spdlog
  GIT_REPOSITORY https://github.com/gabime/spdlog.git
  GIT_TAG        v1.x
)

# ... 其他依赖
```

### BoringSSL 兼容性处理

```cmake
# MinGW 兼容性 patch
if(MINGW)
  # 处理 BoringSSL 汇编优化
  # 自动下载 NASM 汇编器
endif()
```

### 子目录添加

```cmake
add_subdirectory(src)
add_subdirectory(tests)
add_subdirectory(benchmarks)
add_subdirectory(stresses)
```

## src/CMakeLists.txt

负责构建主程序和核心库。

### 静态库构建

```cmake
add_library(prism_static_library STATIC
  ${PRISM_SOURCES}
)

target_include_directories(prism_static_library
  PUBLIC ${CMAKE_SOURCE_DIR}/include
)
```

### 可执行文件构建

```cmake
add_executable(Prism
  main.cpp
)

target_link_libraries(Prism
  PRIVATE prism_static_library
  PRIVATE ${SYSTEM_LIBRARIES}
)
```

### Windows 静态链接配置

```cmake
if(MINGW)
  # 静态链接 MinGW 运行时
  target_link_options(Prism PRIVATE -static)
  target_link_options(Prism PRIVATE -static-libgcc)
  target_link_options(Prism PRIVATE -static-libstdc++)
endif()
```

### 系统库依赖（Windows）

```cmake
if(WIN32)
  set(SYSTEM_LIBRARIES
    ws2_32    # Winsock
    mswsock   # Microsoft Windows Sockets
    crypt32   # Crypto API
  )
endif()
```

## tests/CMakeLists.txt

负责构建测试套件。

### 测试添加函数

```cmake
function(forward_add_test TEST_NAME)
  add_executable(${TEST_NAME} ${TEST_NAME}.cpp)
  target_link_libraries(${TEST_NAME} PRIVATE prism_static_library)
  add_test(NAME ${TEST_NAME} COMMAND ${TEST_NAME})
endfunction()
```

### 测试分组

| 分组 | 测试目标 | 说明 |
|------|----------|------|
| 基础设施 | Trace, Memory, Fault | 核心基础设施测试 |
| 协议 | Http, Socks5, Trojan, Vless, Shadowsocks | 协议编解码测试 |
| 端到端 | E2E, Regression | 集成测试 |
| 多路复用 | Smux, Yamux, MultiplexDuct | 多路复用测试 |
| 加密 | Crypto, Hkdf, Block, Aead | 加密模块测试 |
| 其他 | Session, Connection, Reality 等 | 功能测试 |

### MinGW 大目标文件处理

```cmake
if(MINGW)
  # 避免 "file too big" 错误
  target_compile_options(${TEST_NAME} PRIVATE -Wa,-mbig-obj)
endif()
```

## benchmarks/CMakeLists.txt

负责构建性能基准测试。

### Google Benchmark 集成

```cmake
FetchContent_Declare(
  benchmark
  GIT_REPOSITORY https://github.com/google/benchmark.git
  GIT_TAG        v1.8.0
)

set(BENCHMARK_ENABLE_TESTING OFF)
FetchContent_MakeAvailable(benchmark)
```

### 基准测试列表

| 基准测试 | 测试目标 |
|----------|----------|
| CodecBench | 编解码性能 |
| CryptoBench | 加密算法性能 |
| IOBench | I/O 性能 |
| LatencyBench | 延迟测试 |
| MemoryBench | 内存操作性能 |
| MuxBench | 多路复用性能 |
| MuxConnBench | 多路复用连接性能 |
| ProtocolBench | 协议处理性能 |
| RealityBench | Reality 握手性能 |
| ResourceBench | 资源使用测试 |
| ScaleBench | 扩展性测试 |
| SessionBench | 会话性能 |

### 基准测试构建

```cmake
add_executable(SessionBench SessionBench.cpp)
target_link_libraries(SessionBench
  PRIVATE prism_static_library
  PRIVATE benchmark::benchmark
)
```

## stresses/CMakeLists.txt

负责构建压力测试工具。

### 压力测试列表

| 压力测试 | 测试目标 |
|----------|----------|
| MemoryStress | 内存压力测试 |
| PoolContentionStress | 内存池竞争测试 |
| ArenaOverflowStress | 竞技场溢出测试 |
| MuxStress | 多路复用压力测试 |

### 压力测试构建

```cmake
add_executable(MemoryStress MemoryStress.cpp)
target_link_libraries(MemoryStress PRIVATE prism_static_library)
```

## 相关页面

- [[dev/building/overview|构建系统概述]] — 构建流程概览
- [[dev/building/dependencies|依赖项]] — 外部依赖说明
- [[dev/building/commands|构建命令]] — 常用构建命令
- [[dev/building/options|构建选项]] — 可配置构建选项
- [[dev/testing|测试指南]] — 测试详情
- [[dev/performance/report|性能基准]] — 基准测试结果