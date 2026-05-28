---
layer: dev
title: Prism 性能分析方法
tags: [dev, performance, profiling]
---

# Prism 性能分析方法

本页介绍 Prism 代理服务器的性能分析方法，包括基准测试运行、性能瓶颈定位、火焰图生成。

## 基准测试框架

Prism 使用 Google Benchmark v1.8 进行性能测试，涵盖加密、内存、I/O、延迟、多路复用等多个维度。

### 基准测试清单

| 基准文件 | 测试内容 |
|:---------|:---------|
| `CryptoBench.exe` | AES-GCM、X25519、BLAKE3、HKDF |
| `MemoryBench.exe` | Frame Arena、ThreadLocal、Global Pool |
| `IOBench.exe` | TCP Echo、连接延迟 |
| `LatencyBench.exe` | P50/P90/P99 分布 |
| `MuxBench.exe` | smux/yamux 帧序列化 |
| `ProtocolBench.exe` | HTTP/SOCKS5/Trojan/VLESS 解析 |
| `RealityBench.exe` | Reality 握手分解 |
| `RegressionBench.exe` | 性能回归检查 |

### 运行基准测试

```bash
# 编译启用基准
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_BENCHMARK=ON
cmake --build build_release --config Release

# 运行所有基准
build_release/benchmarks/CryptoBench.exe
build_release/benchmarks/MemoryBench.exe
build_release/benchmarks/IOBench.exe
build_release/benchmarks/LatencyBench.exe
build_release/benchmarks/MuxBench.exe
build_release/benchmarks/ProtocolBench.exe
build_release/benchmarks/RealityBench.exe
build_release/benchmarks/RegressionBench.exe
```

### 输出格式解读

| 字段 | 含义 |
|:-----|:-----|
| 运行次数 | 测试执行的总次数 |
| 每次耗时 | 从开始到结束的实际时间 |
| CPU耗时 | CPU 实际执行时间（不含等待） |
| 速率 | 每秒处理量（Gi/s = GB/s, k/s = 千次/秒） |

## 性能瓶颈定位

### 关键瓶颈指标

| 瓶颈 | 当前值 | 理论极限 | 差距倍数 |
|:-----|:------:|:--------:|:--------:|
| TCP 小包回环 | 20 us | 内存 0.1 us | 200x |
| 连接 P99 | 336 us | <200 us | 1.7x |
| 全局内存池 4T | 3530 ns | <100 ns | 35x |

### 瓶颈根因分析

#### TCP 小包 (20 us vs 理论 0.1 us)

- **根因**: 内核 syscall + socket 缓冲区拷贝
- **占比**: 系统开销 >99%
- **优化**: 内核旁路（io_uring 仅 Linux），Windows 下目标 12-15 us

#### 连接 P99 抖动

- **根因**: 系统调度、内核缓冲区竞争、健康检查阻塞
- **优化**: 快速路径跳过健康检查、预创建机制

#### 全局池多线程竞争

- **根因**: `synchronized_pool` 全局互斥锁
- **优化**: 分片池 (`sharded_pool`)，每个线程独占分片

## 火焰图生成

### Linux 下生成火焰图

```bash
# 安装 perf
sudo apt install linux-perf

# 运行 Prism 服务
./build_release/src/Prism.exe

# 记录性能数据（60 秒）
sudo perf record -g -p $(pgrep Prism) -- sleep 60

# 生成火焰图
sudo perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### Windows 下生成火焰图

Windows 下推荐使用 **Windows Performance Analyzer** 或 **Intel VTune**。

```powershell
# 使用 Windows Performance Recorder
wpr -start GeneralProfile
# 运行 Prism
wpr -stop profile.etl
# 使用 WPA 分析 profile.etl
```

## 性能分析最佳实践

### 1. 建立基线

每次重大修改后运行完整基准套件，记录所有指标。对比历史数据发现退化。

### 2. 关注 P99

P50 可能稳定，但 P99 抖动反映系统级问题。关注延迟分布而非单一均值。

### 3. 内存分配追踪

使用 `MemoryBench.exe` 验证热路径零 malloc。检查 Frame Arena 使用率。

### 4. 加密性能验证

AES-NI 启用与否差异巨大（67x）。每次编译后验证 CryptoBench 输出。

## 相关页面

- [[dev/performance/overview|performance/overview]] — 性能架构概览
- [[dev/performance/tuning|performance/tuning]] — 性能调优指南
- [[dev/performance/report|performance/report]] — 完整基准报告
- [[dev/testing/stress|stress]] — 压力测试方法

## 参考资源

- [Google Benchmark](https://github.com/google/benchmark)
- [FlameGraph](https://github.com/brendangregg/FlameGraph)
- [Linux perf](https://perf.wiki.kernel.org/index.php/Main_Page)