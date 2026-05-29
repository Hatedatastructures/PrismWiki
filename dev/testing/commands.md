---
layer: dev
title: 测试命令
source: tests/CMakeLists.txt
module: testing
type: reference
tags: [testing, ctest, cmake, commands]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing/overview]]"
  - "[[dev/testing/framework]]"
  - "[[dev/testing/writing]]"
---

# 测试命令详解

本文档详细介绍 Prism 项目的测试运行命令，包括 CTest 和单独运行。

## 构建测试

### 配置构建

```bash
# Release 构建（推荐）
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release

# Debug 构建（调试时使用）
cmake -B build_debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build_debug --config Debug
```

### 构建选项

```bash
# 禁用基准测试
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_BENCHMARK=OFF

# 禁用压力测试
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_STRESS=OFF

# 同时禁用两者
cmake -B build_release -DCMAKE_BUILD_TYPE=Release \
    -DPRISM_ENABLE_BENCHMARK=OFF \
    -DPRISM_ENABLE_STRESS=OFF
```

## CTest 命令

### 基本运行

```bash
# 运行所有测试
ctest --test-dir build_release --output-on-failure

# 详细输出模式
ctest --test-dir build_release -V

# 超详细输出
ctest --test-dir build_release -VV
```

### 并行运行

```bash
# 4 线程并行
ctest --test-dir build_release -j 4

# 自动检测 CPU 核数并行
ctest --test-dir build_release -j auto
```

### 选择性运行

```bash
# 运行匹配名称的测试
ctest --test-dir build_release -R Socks5

# 运行正则匹配的测试
ctest --test-dir build_release -R "Http|Socks"

# 排除匹配的测试
ctest --test-dir build_release -E E2E

# 排除多个测试
ctest --test-dir build_release -E "E2E|Regression"
```

### 标签过滤

```bash
# 运行指定标签的测试
ctest --test-dir build_release -L protocol

# 运行多个标签
ctest --test-dir build_release -L "protocol|crypto"

# 排除标签
ctest --test-dir build_release -LE e2e
```

### 列出测试

```bash
# 列出所有测试名称
ctest --test-dir build_release -N

# 显示测试数量
ctest --test-dir build_release --show-only
```

### 超时控制

```bash
# 设置全局超时（秒）
ctest --test-dir build_release --timeout 60

# 设置单个测试超时
ctest --test-dir build_release --timeout 300 -R E2E
```

### 重试机制

```bash
# 失败后重试 3 次
ctest --test-dir build_release --output-on-failure --repeat-until-fail 3

# 直到通过，最多尝试 5 次
ctest --test-dir build_release --repeat-until-pass 5
```

## 单独运行测试

### 基础设施测试

```bash
# 会话测试
build_release/tests/Session.exe

# 连接池测试
build_release/tests/Connection.exe

# 传输测试
build_release/tests/Transmission.exe

# 日志测试
build_release/tests/Trace.exe
```

### 协议测试

```bash
# HTTP 协议
build_release/tests/Http.exe
build_release/tests/HttpParser.exe

# SOCKS5 协议
build_release/tests/Socks5.exe

# Trojan 协议
build_release/tests/Trojan.exe

# VLESS 协议
build_release/tests/Vless.exe

# Shadowsocks 协议
build_release/tests/Shadowsocks.exe
```

### 加密测试

```bash
# 加密模块综合测试
build_release/tests/Crypto.exe

# AEAD 加密
build_release/tests/Aead.exe

# BLAKE3 哈希
build_release/tests/Blake3.exe

# HKDF 密钥派生
build_release/tests/Hkdf.exe

# X25519 密钥交换
build_release/tests/X25519.exe

# 块加密
build_release/tests/Block.exe

# Base64 编码
build_release/tests/Base64Encode.exe
```

### 多路复用测试

```bash
# smux 帧解析
build_release/tests/Smux.exe

# smux 帧构建
build_release/tests/SmuxCraft.exe

# yamux 帧解析
build_release/tests/Yamux.exe

# yamux 帧构建
build_release/tests/YamuxCraft.exe

# 多路复用管道
build_release/tests/MultiplexDuct.exe
```

### 伪装测试

```bash
# Reality TLS
build_release/tests/Reality.exe

# ShadowTLS
build_release/tests/Shadowtls.exe
build_release/tests/ShadowTlsE2E.exe

# Restls
build_release/tests/Restls.exe

# 执行器
build_release/tests/Executor.exe
```

### DNS 测试

```bash
# DNS 缓存
build_release/tests/DnsCache.exe

# DNS 报文
build_release/tests/DnsPacket.exe

# DNS 规则
build_release/tests/DnsRules.exe
```

### 其他测试

```bash
# 错误码
build_release/tests/Fault.exe

# 异常
build_release/tests/Exception.exe

# 内存竞技场
build_release/tests/MemoryArena.exe

# 配置解析
build_release/tests/Config.exe

# 负载均衡
build_release/tests/Balancer.exe

# 账户目录
build_release/tests/AccountDirectory.exe

# 协议识别
build_release/tests/Recognition.exe

# 管道原语
build_release/tests/PipelinePrimitives.exe

# 端到端
build_release/tests/E2E.exe

# 回归
build_release/tests/Regression.exe
```

## 并发测试运行

并发测试需要同时运行 server 和 client：

```bash
# 终端 1: 启动 server
build_release/tests/concurrency/server.exe

# 终端 2: 运行 client
build_release/tests/concurrency/client.exe
```

或者使用脚本自动化：

```bash
# Windows (PowerShell)
Start-Process build_release/tests/concurrency/server.exe
Start-Sleep -Seconds 2
build_release/tests/concurrency/client.exe

# Linux (Bash)
build_release/tests/concurrency/server &
sleep 2
build_release/tests/concurrency/client
```

## 基准测试运行

基准测试需要启用 `PRISM_ENABLE_BENCHMARK=ON`：

```bash
# 编解码基准
build_release/benchmarks/CodecBench.exe

# 加密基准
build_release/benchmarks/CryptoBench.exe

# I/O 基准
build_release/benchmarks/IOBench.exe

# 延迟基准
build_release/benchmarks/LatencyBench.exe

# 内存基准
build_release/benchmarks/MemoryBench.exe

# 多路复用基准
build_release/benchmarks/MuxBench.exe
build_release/benchmarks/MuxConnBench.exe

# 协议基准
build_release/benchmarks/ProtocolBench.exe

# Reality 基准
build_release/benchmarks/RealityBench.exe

# 资源基准
build_release/benchmarks/ResourceBench.exe

# 扩展性基准
build_release/benchmarks/ScaleBench.exe

# 会话基准
build_release/benchmarks/SessionBench.exe
```

基准测试输出示例：

```
Benchmark                    Time             CPU   Iterations
-------------------------------------------------------------
BM_AesGcmEncrypt/16         123 ns         123 ns      5678900
BM_AesGcmEncrypt/1024      1234 ns        1234 ns       567890
BM_AesGcmEncrypt/65536   123456 ns       123456 ns        5678
```

## 压力测试运行

压力测试需要启用 `PRISM_ENABLE_STRESS=ON`：

```bash
# 内存压力
build_release/stresses/MemoryStress.exe

# 池竞争压力
build_release/stresses/PoolContentionStress.exe

# 竞技场溢出压力
build_release/stresses/ArenaOverflowStress.exe

# 多路复用压力
build_release/stresses/MuxStress.exe
```

## 测试输出解读

### 成功输出

```
[Session] PASS: session created
[Session] PASS: session started
[Session] PASS: session closed
[Session] Results: 3 passed, 0 failed
```

退出码：0

### 失败输出

```
[Socks5] PASS: version negotiation
[Socks5] FAIL: authentication (expected success, got failure)
[Socks5] PASS: CONNECT command
[Socks5] Results: 2 passed, 1 failed
```

退出码：1

### CTest 汇总

```
100% tests passed, 0 tests failed out of 44

Total Test time (real) =   12.34 sec
```

## 调试技巧

### 使用调试器

```bash
# GDB
gdb build_debug/tests/Socks5.exe
run

# Visual Studio
devenv /debugexe build_debug/tests/Socks5.exe
```

### 日志级别

```cpp
// 在测试中设置日志级别
psm::trace::set_level(psm::trace::level::debug);
```

### 单步断点

```cpp
// 在测试代码中添加断点位置
runner.Check(condition, "checkpoint 1");
if (!condition) {
    // 断点在这里，检查变量
}
```

## CI/CD 测试

GitHub Actions 自动运行测试：

```yaml
# .github/workflows/build.yml
- name: Run Tests
  run: ctest --test-dir build --output-on-failure -j 4
```

## 命令速查表

| 命令 | 说明 |
|------|------|
| `ctest --test-dir build` | 运行所有测试 |
| `ctest --test-dir build -j 4` | 4 线程并行 |
| `ctest --test-dir build -R Name` | 运行匹配测试 |
| `ctest --test-dir build -E Name` | 排除测试 |
| `ctest --test-dir build -V` | 详细输出 |
| `ctest --test-dir build -N` | 列出测试 |
| `build/tests/Test.exe` | 单独运行 |

## 相关链接

- [[dev/testing/overview]] — 测试体系概述
- [[dev/testing/framework]] — TestRunner 框架
- [[dev/testing/writing]] — 测试编写指南
- [[dev/testing/benchmark]] — 基准测试
- [[dev/testing/stress]] — 压力测试
- [[dev/testing/concurrency]] — 并发测试