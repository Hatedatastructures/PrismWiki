---
layer: dev
title: 基准测试
source: I:/code/Prism/benchmarks/
module: testing
type: reference
tags: [testing, benchmark, performance, google-benchmark]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing/overview]]"
  - "[[dev/testing/commands]]"
  - "[[dev/performance/report|performance/benchmark]]"
---

# 基准测试体系

Prism 包含 12 个基准测试，使用 Google Benchmark 库测量各模块性能。

## 基准测试列表

| 基准测试 | 文件 | 覆盖范围 |
|----------|------|----------|
| CodecBench | CodecBench.cpp | 编解码性能 |
| CryptoBench | CryptoBench.cpp | 加密算法性能 |
| IOBench | IOBench.cpp | I/O 操作性能 |
| LatencyBench | LatencyBench.cpp | 尾延迟测量 |
| MemoryBench | MemoryBench.cpp | 内存分配性能 |
| MuxBench | MuxBench.cpp | 多路复用帧处理 |
| MuxConnBench | MuxConnBench.cpp | 多路复用连接性能 |
| ProtocolBench | ProtocolBench.cpp | 协议处理性能 |
| RealityBench | RealityBench.cpp | Reality TLS 性能 |
| ResourceBench | ResourceBench.cpp | 资源使用测量 |
| ScaleBench | ScaleBench.cpp | 扩展性测试 |
| SessionBench | SessionBench.cpp | 会话处理性能 |

## 构建配置

基准测试需要启用 `PRISM_ENABLE_BENCHMARK=ON`（默认开启）：

```bash
# 构建包含基准测试
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release

# 禁用基准测试
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_BENCHMARK=OFF
```

`benchmarks/CMakeLists.txt` 自动获取 Google Benchmark：

```cmake
FetchContent_Declare(
    benchmark
    URL https://github.com/google/benchmark/archive/refs/tags/v1.9.5.tar.gz
)
FetchContent_MakeAvailable(benchmark)
```

## 运行基准测试

### 单独运行

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

### Google Benchmark 参数

```bash
# 运行指定基准
build_release/benchmarks/CryptoBench.exe --benchmark_filter=AES

# 设置迭代次数
build_release/benchmarks/CryptoBench.exe --benchmark_min_time=5.0

# 输出 JSON 格式
build_release/benchmarks/CryptoBench.exe --benchmark_format=json

# 输出 CSV 格式
build_release/benchmarks/CryptoBench.exe --benchmark_format=csv

# 显示聚合统计
build_release/benchmarks/CryptoBench.exe --benchmark_report_aggregates_only

# 禁用 CPU 频率缩放警告
build_release/benchmarks/CryptoBench.exe --benchmark_no_warnings
```

## CodecBench — 编解码基准

测量协议编解码性能：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_HttpRequestEncode/64          45 ns       45 ns    15000000
BM_HttpRequestEncode/1024       450 ns      450 ns     1500000
BM_Socks5UdpHeaderEncode        12 ns       12 ns    60000000
BM_TrojanRequestEncode          55 ns       55 ns    13000000
BM_VlessRequestEncode           48 ns       48 ns    15000000
BM_SmuxFrameEncode              15 ns       15 ns    45000000
BM_YamuxFrameEncode             18 ns       18 ns    40000000
```

测试内容：
- HTTP 请求构建
- SOCKS5 UDP 头编码
- Trojan 请求构建
- VLESS 请求构建
- smux 帧编码
- yamux 帧编码

## CryptoBench — 加密基准

测量加密算法性能：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_Aes128GcmEncrypt/16          85 ns       85 ns     8000000
BM_Aes128GcmEncrypt/1024       820 ns      820 ns      800000
BM_Aes128GcmEncrypt/16384    13000 ns     13000 ns       54000
BM_Aes256GcmEncrypt/16          95 ns       95 ns     7000000
BM_Chacha20Poly1305/16         150 ns      150 ns     4800000
BM_Blake3Hash/16                12 ns       12 ns    60000000
BM_Blake3Hash/1024             120 ns      120 ns     5800000
BM_HkdfExtract                 180 ns      180 ns     4000000
BM_X25519KeyExchange          3500 ns     3500 ns      200000
```

测试内容：
- AES-128-GCM 加密（16B, 1KB, 16KB）
- AES-256-GCM 加密
- ChaCha20-Poly1305 加密
- BLAKE3 哈希
- HKDF 密钥派生
- X25519 密钥交换

## IOBench — I/O 基准

测量 I/O 操作性能：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_BufferRead/64                10 ns       10 ns    70000000
BM_BufferRead/1024              30 ns       30 ns    23000000
BM_BufferWrite/64               12 ns       12 ns    58000000
BM_BufferWrite/1024             35 ns       35 ns    20000000
BM_SocketRead/64               500 ns      500 ns     1400000
BM_SocketWrite/64              450 ns      450 ns     1600000
```

测试内容：
- 缓冲区读取
- 缓冲区写入
- Socket 读取（模拟）
- Socket 写入（模拟）

## LatencyBench — 延迟基准

测量 P50/P90/P99 尾延迟：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_SessionLatency/P50          120 us      120 us       12000
BM_SessionLatency/P90          450 us      450 us        3100
BM_SessionLatency/P99         1200 us     1200 us        1200
BM_MuxConnLatency/P50          85 us       85 us       16000
BM_MuxConnLatency/P90         280 us      280 us        5100
BM_MuxConnLatency/P99         750 us      750 us        1900
```

测试内容：
- 会话建立延迟分布
- 多路复用连接延迟分布
- 协议处理延迟分布

## MemoryBench — 内存基准

测量内存分配性能：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_FrameArenaAllocate/16         8 ns        8 ns    80000000
BM_FrameArenaAllocate/1024      12 ns       12 ns    60000000
BM_FrameArenaAllocate/65536     45 ns       45 ns    15000000
BM_FrameArenaReset               2 ns        2 ns   300000000
BM_GlobalPoolAllocate           15 ns       15 ns    45000000
BM_PmrVectorPush/100            30 ns       30 ns    23000000
```

测试内容：
- 帧竞技场分配
- 帧竞技场重置
- 全局内存池分配
- PMR 容器操作

## MuxBench — 多路复用帧基准

测量帧处理性能：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_SmuxSynParse                  5 ns        5 ns   140000000
BM_SmuxPshParse                  8 ns        8 ns    80000000
BM_SmuxFinParse                  5 ns        5 ns   140000000
BM_YamuxDataParse               10 ns       10 ns    70000000
BM_YamuxWindowParse              8 ns        8 ns    88000000
BM_MuxDuctEstablish            850 ns      850 ns      820000
```

测试内容：
- smux 帧解析（SYN/PSH/FIN）
- yamux 帧解析（Data/Window）
- 多路复用管道建立

## MuxConnBench — 多路复用连接基准

测量多路复用连接吞吐量：

```
Benchmark                              Time         CPU   Iterations
-------------------------------------------------------------------
BM_MuxConnectionEstablish/1C1S        500 us      500 us        2800
BM_MuxConnectionEstablish/4C64S      2000 us     2000 us         700
BM_MuxConnectionEstablish/1C256S     8000 us     8000 us         175
BM_MuxStreamDataTransfer/1KB         150 ns      150 ns     4500000
BM_MuxStreamDataTransfer/64KB       2000 ns     2000 ns       350000
```

测试内容：
- 连接/流组合（UCUS、4C64S、1C256S）
- 流数据传输吞吐量

## ProtocolBench — 协议基准

测量协议处理性能：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_HttpHandlerProcess           250 us      250 us        5600
BM_Socks5HandlerProcess         180 us      180 us        7800
BM_TrojanHandlerProcess         120 us      120 us       12000
BM_VlessHandlerProcess          130 us      130 us       11000
BM_ShadowsocksHandlerProcess    150 us      150 us        9300
BM_ProbeDetect                   35 ns       35 ns    20000000
BM_ClientHelloParse             150 ns      150 ns     4700000
```

测试内容：
- 各协议处理器处理时间
- 协议探测时间
- ClientHello 解析时间

## RealityBench — Reality TLS 基准

测量 Reality 伪装性能：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_RealityShortIdGenerate        45 ns       45 ns    15000000
BM_RealityKeyDerive              85 ns       85 ns     8000000
BM_RealityVerifyTime             12 ns       12 ns    60000000
BM_RealityHandshakeTotal        3500 us     3500 us         400
BM_RealityActiveProbeResponse   200 us      200 us        7000
```

测试内容：
- Short ID 生成
- 密钥派生
- 时间验证
- 完整握手时间
- 主动探测响应

## ResourceBench — 资源基准

测量资源使用：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_SessionMemoryUsage            2 KB        2 KB      500000
BM_ConnectionPoolMemory         15 KB       15 KB      100000
BM_MuxStreamMemory               4 KB        4 KB      250000
BM_DnsCacheMemory/100           10 KB       10 KB      100000
BM_DnsCacheMemory/1000         100 KB      100 KB       10000
```

测试内容：
- 会话内存占用
- 连接池内存占用
- 多路复用流内存占用
- DNS 缓存内存占用

## ScaleBench — 扩展性基准

测量并发扩展性：

```
Benchmark                              Time         CPU   Iterations
-------------------------------------------------------------------
BM_ConcurrentSessions/1               1.2 ms      1.2 ms        1000
BM_ConcurrentSessions/10              8.5 ms      8.5 ms         150
BM_ConcurrentSessions/100            85 ms       85 ms          15
BM_ConcurrentSessions/1000          850 ms      850 ms          2
BM_WorkerScaling/1                    1.0 ms      1.0 ms        1400
BM_WorkerScaling/4                    0.3 ms      0.3 ms        4500
BM_WorkerScaling/8                    0.2 ms      0.2 ms        6000
```

测试内容：
- 并发会话处理
- Worker 数量扩展
- 连接池扩展

## SessionBench — 会话基准

测量会话生命周期性能：

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_SessionCreate                 500 us      500 us        2800
BM_SessionHandshake              350 us      350 us        4000
BM_SessionDataTransfer/1KB      120 us      120 us        9300
BM_SessionDataTransfer/64KB    1500 us     1500 us         730
BM_SessionClose                  50 us       50 us       28000
BM_SessionFullLifecycle        1000 us     1000 us        1400
```

测试内容：
- 会话创建
- 协议握手
- 数据传输
- 会话关闭
- 完整生命周期

## 输出格式

### 默认输出

```
Benchmark                         Time         CPU   Iterations
--------------------------------------------------------------
BM_AesGcmEncrypt/16             85 ns       85 ns     8000000
BM_AesGcmEncrypt/1024          820 ns      820 ns      800000
BM_AesGcmEncrypt/16384       13000 ns    13000 ns       54000
BM_AesGcmEncrypt_BigO           0.80 N      0.80 N
BM_AesGcmEncrypt_RMS              1 %         1 %
```

### JSON 输出

```bash
build_release/benchmarks/CryptoBench.exe --benchmark_format=json > crypto_results.json
```

### CSV 输出

```bash
build_release/benchmarks/CryptoBench.exe --benchmark_format=csv > crypto_results.csv
```

## 性能分析

使用基准测试结果进行性能分析：

```bash
# 比较两次运行
./compare_benchmarks.py old_results.json new_results.json

# 生成报告
./generate_report.py crypto_results.json
```

## 相关链接

- [[dev/testing/overview]] — 测试体系概述
- [[dev/testing/commands]] — 测试命令详解
- [[dev/performance/report|performance/benchmark]] — 性能基准报告
- [[dev/testing/stress]] — 压力测试体系