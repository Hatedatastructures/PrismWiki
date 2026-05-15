---
title: Prism 测试体系
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [testing, proxy-server, benchmark, cpp]
sources:
  - I:/code/Prism/CLAUDE.md
confidence: high
---

# Prism 测试体系

Prism 采用自定义测试框架，包含单元测试、基准测试、压力测试和并发测试。

## 测试框架

测试为独立可执行文件，无外部测试框架。所有测试共用 `tests/common/TestRunner.hpp` 中的 `psm::testing::TestRunner` 类。

### 测试框架功能

- `TestRunner(tag)` — 创建测试运行器，tag 用于区分测试模块
- `Check(condition, message)` — 检查条件，自动记录 pass/fail
- `LogPass(message)` / `LogFail(message)` — 手动记录测试结果
- `Summary()` — 输出结果汇总，返回退出码（0=全部通过，1=存在失败）

## 测试目标

共 42 个测试目标，另 2 个并发测试：

### 基础设施测试

- Session、Connection、Transmission、Trace

### 协议测试

- Http、Socks5、Trojan、Vless、Shadowsocks

### 端到端集成测试

- E2E

### 回归测试

- Regression

### 单元测试

- HttpParser、Smux、SmuxCraft、Yamux、YamuxCraft、MultiplexDuct
- Crypto、Hkdf、Block、Aead、Reality、Blake3、X25519
- DnsPacket、DnsRules、Fault、Exception
- ProtocolAnalysis、ProtocolToString、Balancer、DnsCache
- AccountDirectory、Shadowtls、Restls、Config、Base64Encode
- MemoryArena、PipelinePrimitives、Recognition、Executor、FaultHandling

## 测试文件位置

- 测试文件：`tests/*.cpp`
- 并发测试：`tests/concurrency/server.cpp`、`tests/concurrency/client.cpp`

## 运行测试

```bash
# 运行所有测试
ctest --test-dir build_release --output-on-failure

# 运行单个测试
build_release/tests/Session.exe
build_release/tests/Socks5.exe
build_release/tests/Trojan.exe
```

## 并发测试

位于 `tests/concurrency/`，需要同时运行 server 和 client：

```bash
# 终端 1：启动 server
build_release/tests/concurrency/server.exe

# 终端 2：运行 client
build_release/tests/concurrency/client.exe
```

## 基准测试

使用 Google Benchmark 库，12 个基准测试：

- CodecBench — 编解码性能
- MemoryBench — 内存分配性能
- MuxBench — 多路复用性能
- CryptoBench — 加密性能
- ProtocolBench — 协议处理性能
- IOBench — I/O 性能
- MuxConnBench — 多路复用连接性能
- SessionBench — 会话性能
- RealityBench — Reality TLS 性能
- ScaleBench — 扩展性测试
- ResourceBench — 资源使用测试
- LatencyBench — 延迟测试

### 运行基准测试

```bash
# 运行基准测试（若启用 PRISM_ENABLE_BENCHMARK=ON）
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

## 压力测试

4 个压力测试：

- MemoryStress — 内存压力测试
- PoolContentionStress — 内存池竞争测试
- ArenaOverflowStress — Arena 溢出测试
- MuxStress — 多路复用压力测试

### 运行压力测试

```bash
# 运行压力测试（若启用 PRISM_ENABLE_STRESS=ON）
build_release/stresses/MemoryStress.exe
build_release/stresses/PoolContentionStress.exe
build_release/stresses/ArenaOverflowStress.exe
build_release/stresses/MuxStress.exe
```

## 构建选项

- `PRISM_ENABLE_BENCHMARK=ON/OFF` — 启用/禁用基准测试（默认：ON）
- `PRISM_ENABLE_STRESS=ON/OFF` — 启用/禁用压力测试工具（默认：ON）

## 相关页面

- [[agent]] — 项目主页
- [[docs/agent/architecture]] — 架构设计
- [[docs/agent/configuration]] — 配置详解
- [[dev/tcp]] — 测试中的传输层
- [[docs/protocol/proxy-protocols]] — 协议测试
