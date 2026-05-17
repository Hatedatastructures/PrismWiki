---
layer: dev
---

# Prism 性能基准测试报告

> 测试日期: 2026-05-06
> 测试平台: Intel Core i9-13900K (24核/32线程 @ 2995 MHz)
> 内存配置: DDR4 64GB
> 编译环境: MinGW GCC 15.2.0, C++23, Release (-O3)
> BoringSSL: AES-NI 汇编已启用（MinGW patch）

---

## 一、协议握手性能

协议握手延迟直接影响连接建立速度。SS2022 最快（纯内存操作），其他协议均需 TCP 往返。

### 握手延迟

| 测试项 | 运行次数 | 每次耗时 | CPU耗时 | 速率 | 说明 |
|:-------|:--------:|:--------:|:-------:|:----:|:-----|
| HTTP握手 | 4480 | 170 us | 164 us | 6.10 k/s | |
| SOCKS5握手 | 4480 | 146 us | 146 us | 6.83 k/s | |
| Trojan握手 | 5600 | 131 us | 131 us | 7.63 k/s | |
| VLESS握手 | 5600 | 138 us | 140 us | 7.17 k/s | |
| SS2022握手 | 560000 | 1.35 us | 1.37 us | 731.4 k/s | 纯内存，无网络 |

### 与竞品对比

| 实现名称 | 每次耗时 | 说明 |
|:---------|:--------:|:-----|
| Prism SS2022 | 1.35 us | 本测试 |
| shadowsocks-rust | 5-10 us | PSK 解析 + 密钥派生 |
| Prism Trojan | 131 us | 本测试 |
| trojan-gfw | 150-200 us | 社区反馈 |
| Prism VLESS | 138 us | 本测试 |

---

## 二、隧道吞吐量

| 测试项 | 运行次数 | 每次耗时 | 速率 |
|:-------|:--------:|:--------:|:----:|
| 隧道传输/16KB | 49778 | 21.8 us | 2.86 Gi/s |
| 隧道传输/64KB | 56000 | 33.6 us | 6.16 Gi/s |
| 隧道传输/128KB | 18667 | 49.5 us | 7.11 Gi/s |

**分析**: TCP 回环涉及内核上下文切换 + socket 缓冲区拷贝，大包场景 TCP 吞吐达 7 Gi/s，接近内存带宽极限。

---

## 三、加密性能 (AEAD)

### AES-GCM 连续吞吐量 (64KB payload) — AES-NI 已启用

| 测试项 | 运行次数 | 每次耗时 | 速率 |
|:-------|:--------:|:--------:|:----:|
| AES-128-GCM加密 | 186667 | 3767 ns | **16.2 Gi/s** |
| AES-128-GCM解密 | 194783 | 3610 ns | **16.9 Gi/s** |
| AES-256-GCM加密 | 112000 | 4464 ns | **13.7 Gi/s** |
| AES-256-GCM解密 | 160000 | 4199 ns | **14.5 Gi/s** |

**性能提升**: AES-256-GCM 从 205 Mi/s → 13.7 Gi/s，**提升 67 倍**。

---

## 四、密钥交换 (X25519)

| 测试项 | 运行次数 | 每次耗时 | CPU耗时 |
|:-------|:--------:|:--------:|:-------:|
| X25519密钥生成 | 49778 | 15.7 us | 15.7 us |
| X25519公钥派生 | 44800 | 15.0 us | 15.0 us |
| X25519密钥交换 | 32000 | **21.4 us** | 21.5 us |

**性能提升**: X25519 密钥交换从 81.5 us → 21.4 us，**提升 3.8 倍**。

---

## 五、密钥派生性能

| 测试项 | 运行次数 | 每次耗时 | 速率 | 说明 |
|:-------|:--------:|:--------:|:----:|:-----|
| BLAKE3密钥派生 | 4072727 | 165 ns | 185 Mi/s | SS2022 EK 派生 |
| HKDF Extract | 4977778 | 151 ns | 202.5 Mi/s | TLS 1.3 extract |
| HKDF Expand | 3733333 | 193 ns | 158.5 Mi/s | TLS 1.3 expand |

---

## 六、内存分配性能

| 测试项 | 运行次数 | 每次耗时 | 说明 |
|:-------|:--------:|:--------:|:-----|
| Frame Arena | 112000000 | 9.24 ns | 最快，无竞争 |
| ThreadLocal Pool | 37333333 | 17.7 ns | 无竞争 |
| Global Pool | 5600000 | 119 ns | 有竞争 |

### 多线程竞争

| 测试项 | 每次耗时 | 说明 |
|:-------|:--------:|:-----|
| Global Pool 1线程 | 142 ns | baseline |
| Global Pool 4线程 | 3767 ns | 26.5x slower |
| ThreadLocal Pool4 | 37.5 ns | 103x faster vs Global |

---

## 七、瓶颈分析

| 模块名称 | 当前性能 | 理论极限 | 差距 | 原因 |
|:---------|:--------:|:--------:|:----:|:-----|
| TCP 小包回环 | 20 us | 内存 0.1us | 200x | 系统开销 |
| 连接 P99 | 336 us | <200 us | 1.7x | 健康检查阻塞 |
| 全局内存池 4T | 3530 ns | <100 ns | 35x | 锁竞争 |

### 已验证优化

| 优化项 | 优化前 | 优化后 | 提升 |
|:-------|:------:|:------:|:----:|
| AES-256-GCM | 205 Mi/s | 13.7 Gi/s | 67x |
| X25519 密钥交换 | 81.5 us | 21.4 us | 3.8x |

---

## 八、测试方法

使用 Google Benchmark v1.8，运行命令：

```bash
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_BENCHMARK=ON
cmake --build build_release --config Release

build_release/benchmarks/CryptoBench.exe
build_release/benchmarks/MemoryBench.exe
build_release/benchmarks/IOBench.exe
build_release/benchmarks/LatencyBench.exe
build_release/benchmarks/RegressionBench.exe
```

---

## 相关页面

- [[performance/overview]] — 性能架构概览
- [[performance/tuning]] — 性能调优指南
- [[performance/profiling]] — 性能分析方法
- [[benchmark]] — 基准测试设计

## 参考资源

- [WireGuard Performance](https://www.wireguard.com/performance/)
- [Redis Benchmarks](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)
- [Google Benchmark](https://github.com/google/benchmark)

---

*报告生成: 2026-05-06*
