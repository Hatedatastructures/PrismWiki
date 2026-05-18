---
title: 构建命令
created: 2026-05-17
updated: 2026-05-17
type: reference
tags: [commands, build, cmake, testing, benchmark]
related:
  - "[[dev/building/overview]]"
  - "[[dev/building/cmake]]"
  - "[[dev/building/options]]"
  - "[[dev/testing/testing|testing]]"
  - "[[performance/benchmark]]"
sources:
  - I:/code/Prism/CLAUDE.md
confidence: high
layer: dev
---

# 构建命令

本文档列出 Prism 项目常用的构建命令，包括配置、编译、测试和运行。

## 基础构建

### Release 构建（生产/性能测试）

```bash
# 配置 Release 构建
cmake -B build_release -DCMAKE_BUILD_TYPE=Release

# 编译
cmake --build build_release --config Release
```

**优化选项**：
- `-O3` — 最高优化级别
- `-ffunction-sections` — 函数分段（链接时优化）
- `-fdata-sections` — 数据分段（链接时优化）

### Debug 构建（开发调试）

```bash
# 配置 Debug 构建
cmake -B build_debug -DCMAKE_BUILD_TYPE=Debug

# 编译
cmake --build build_debug --config Debug
```

**Debug 特性**：
- 保留调试符号
- 禁用优化
- 启用断言检查
- 详细错误信息

### 清理构建

```bash
# 清理构建目录
cmake --build build_release --target clean

# 完全重新构建
rm -rf build_release
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release
```

## 测试命令

### 运行所有测试

```bash
# 使用 ctest 运行所有测试
ctest --test-dir build_release --output-on-failure
```

**ctest 常用选项**：
| 选项 | 说明 |
|------|------|
| `--output-on-failure` | 失败时输出详情 |
| `-V` | 详细输出 |
| `-j N` | 并行运行 N 个测试 |
| `-R <regex>` | 运行匹配正则的测试 |
| `-E <regex>` | 排除匹配正则的测试 |

### 运行单个测试

```bash
# 运行 Session 测试
build_release/tests/Session.exe

# 运行 Socks5 协议测试
build_release/tests/Socks5.exe

# 运行 Trojan 协议测试
build_release/tests/Trojan.exe
```

### 完整测试列表

| 测试名称 | 测试目标 |
|----------|----------|
| Session | 会话生命周期 |
| Connection | 连接管理 |
| Transmission | 数据传输 |
| Trace | 日志系统 |
| Http | HTTP 协议 |
| Socks5 | SOCKS5 协议 |
| Trojan | Trojan 协议 |
| Vless | VLESS 协议 |
| Shadowsocks | SS2022 协议 |
| E2E | 端到端集成 |
| Regression | 回归测试 |
| HttpParser | HTTP 解析器 |
| Smux | SMUX 多路复用 |
| SmuxCraft | SMUX 编解码 |
| Yamux | Yamux 多路复用 |
| YamuxCraft | Yamux 编解码 |
| MultiplexDuct | 多路复用管道 |
| Crypto | 加密模块 |
| Hkdf | HKDF 密钥派生 |
| Block | 分组密码 |
| Aead | AEAD 加密 |
| Reality | Reality 握手 |
| Blake3 | BLAKE3 哈希 |
| X25519 | X25519 密钥交换 |
| DnsPacket | DNS 报文解析 |
| DnsRules | DNS 规则匹配 |
| Fault | 错误码 |
| Exception | 异常层次 |
| ProtocolAnalysis | 协议分析 |
| ProtocolToString | 协议字符串转换 |
| Balancer | 负载均衡器 |
| DnsCache | DNS 缓存 |
| AccountDirectory | 账户目录 |
| Shadowtls | ShadowTLS 协议 |
| Restls | Restls 协议 |
| Config | 配置解析 |
| Base64Encode | Base64 编码 |
| MemoryArena | 内存竞技场 |
| PipelinePrimitives | 管道原语 |
| Recognition | 协议识别 |
| Executor | 执行器 |
| FaultHandling | 错误处理 |

### 并发测试

并发测试需要同时运行 server 和 client：

```bash
# 终端 1: 启动 server
build_release/tests/concurrency/server.exe

# 终端 2: 运行 client
build_release/tests/concurrency/client.exe
```

## 基准测试

基准测试需要启用 `PRISM_ENABLE_BENCHMARK=ON`（默认启用）。

### 运行基准测试

```bash
# 运行所有基准测试
build_release/benchmarks/CodecBench.exe
build_release/benchmarks/CryptoBench.exe
build_release/benchmarks/IOBench.exe
build_release/benchmarks/LatencyBench.exe
build_release/benchmarks/MemoryBench.exe
build_release/benchmarks/MuxBench.exe
build_release/benchmarks/MuxConnBench.exe
build_release/benchmarks/ProtocolBench.exe
build_release/benchmarks/RealityBench.exe
build_release/benchmarks/ResourceBench.exe
build_release/benchmarks/ScaleBench.exe
build_release/benchmarks/SessionBench.exe
```

### 基准测试分类

| 基准测试 | 测试目标 | 关键指标 |
|----------|----------|----------|
| CodecBench | 编解码性能 | 吞吐量 |
| CryptoBench | 加密算法 | 加解密速度 |
| IOBench | I/O 性能 | 读写吞吐 |
| LatencyBench | 延迟 | P50/P99 延迟 |
| MemoryBench | 内存操作 | 分配/释放速度 |
| MuxBench | 多路复用 | 流创建/销毁 |
| MuxConnBench | 多路复用连接 | 连接吞吐 |
| ProtocolBench | 协议处理 | 请求/秒 |
| RealityBench | Reality 握手 | 握手延迟 |
| ResourceBench | 资源使用 | 内存/CPU |
| ScaleBench | 扩展性 | 并发连接数 |
| SessionBench | 会话处理 | 会话吞吐 |

## 压力测试

压力测试需要启用 `PRISM_ENABLE_STRESS=ON`（默认启用）。

### 运行压力测试

```bash
# 内存压力测试
build_release/stresses/MemoryStress.exe

# 内存池竞争测试
build_release/stresses/PoolContentionStress.exe

# 竞技场溢出测试
build_release/stresses/ArenaOverflowStress.exe

# 多路复用压力测试
build_release/stresses/MuxStress.exe
```

## 运行代理服务器

```bash
# 运行代理服务器（使用默认配置）
build_release/src/Prism.exe

# 指定配置文件
build_release/src/Prism.exe /path/to/configuration.json
```

**配置文件查找顺序**：
1. 命令行参数指定的路径
2. 可执行文件同目录下的 `configuration.json`

## 常用命令组合

### 完整发布流程

```bash
# 1. 清理旧构建
rm -rf build_release

# 2. 配置 Release 构建
cmake -B build_release -DCMAKE_BUILD_TYPE=Release

# 3. 编译
cmake --build build_release --config Release

# 4. 运行所有测试
ctest --test-dir build_release --output-on-failure

# 5. 运行基准测试（可选）
build_release/benchmarks/SessionBench.exe

# 6. 部署
cp build_release/src/Prism.exe /path/to/deploy/
```

### 快速开发迭代

```bash
# 增量编译 Debug 版本
cmake --build build_debug --config Debug

# 运行相关测试
build_debug/tests/Session.exe
build_debug/tests/Socks5.exe

# 运行服务器（调试）
build_debug/src/Prism.exe
```

## 相关页面

- [[dev/building/overview|构建系统概述]] — 构建流程概览
- [[dev/building/cmake|CMake 构建结构]] — CMake 配置详解
- [[dev/building/options|构建选项]] — 可配置构建选项
- [[dev/testing/testing|测试指南]] — 测试详情
- [[performance/benchmark|性能基准]] — 基准测试结果
- [[dev/testing/stress|压力测试]] — 压力测试详情