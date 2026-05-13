---
title: 基准测试详解
tags: [benchmark, performance, crypto, codec, memory, mux, latency, scale]
source:
  - I:/code/Prism/benchmarks/CodecBench.cpp
  - I:/code/Prism/benchmarks/CryptoBench.cpp
  - I:/code/Prism/benchmarks/IOBench.cpp
  - I:/code/Prism/benchmarks/LatencyBench.cpp
  - I:/code/Prism/benchmarks/MemoryBench.cpp
  - I:/code/Prism/benchmarks/MuxBench.cpp
  - I:/code/Prism/benchmarks/MuxConnBench.cpp
  - I:/code/Prism/benchmarks/ProtocolBench.cpp
  - I:/code/Prism/benchmarks/RealityBench.cpp
  - I:/code/Prism/benchmarks/ResourceBench.cpp
  - I:/code/Prism/benchmarks/ScaleBench.cpp
  - I:/code/Prism/benchmarks/SessionBench.cpp
---

# 基准测试详解

Prism 提供 12 个基准测试，基于 Google Benchmark 库，覆盖编解码、加密、I/O、延迟、内存、多路复用、协议、伪装、资源、扩展性和会话等维度。

## 构建与运行

```bash
# 构建（需启用 PRISM_ENABLE_BENCHMARK=ON，默认已启用）
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_BENCHMARK=ON
cmake --build build_release --config Release

# 运行单个基准测试
build_release/benchmarks/CodecBench.exe
build_release/benchmarks/CryptoBench.exe
# ... 以此类推
```

---

## CodecBench — 协议编解码

测量 HTTP 代理请求解析、DNS 报文处理、加密工具、DNS 规则引擎的性能。

| 测试项 | 关键指标 |
|--------|----------|
| HTTP GET/CONNECT/POST 解析 | ops/s，POST 支持 0~4096 字节 body |
| HTTP 相对路径提取 | ops/s |
| Trojan 凭据解析（56 字节 SHA224） | ops/s，含无效凭据测试 |
| DNS 查询构建/打包/解包 | ops/s |
| DNS IP 提取 / TTL 计算 | ops/s |
| SHA224 短/长输入哈希 | ops/s |
| Base64 短/长解码 | ops/s |
| 凭据规范化（明文/已哈希） | ops/s |
| 域名前缀树查找（精确/通配符/未命中） | ops/s |
| 大规模 DomainTrie（100/1K/10K 规则） | ops/s |
| DNS 规则引擎匹配 | ops/s |
| SS2022 PSK 解码 | ops/s |

---

## CryptoBench — 加密操作

测量 AEAD、BLAKE3、X25519、HKDF 和 Salt Pool 的性能。

| 测试项 | 关键指标 |
|--------|----------|
| AEAD 持续加密/解密 (AES-128/256-GCM, 64KB) | MB/s |
| AEAD 多 payload 大小 (16B~64KB) | MB/s |
| BLAKE3 密钥派生 | ops/s |
| X25519 密钥生成/公钥派生/密钥交换 | ops/s |
| HKDF Extract/Expand/ExpandLabel | ops/s |
| Salt Pool 重放保护 (check_and_insert) | ops/s |

---

## IOBench — TCP/UDP 网络 I/O

测量真实 TCP socket 回环吞吐量（echo 模式），作为网络层基线。

| 测试项 | 关键指标 |
|--------|----------|
| TCP Echo 64B / 1KB / 16KB / 64KB / 128KB | MB/s |
| Memory Copy 64B / 1KB / 16KB / 64KB / 128KB | ns（对照组） |

TCP 测试使用 `pipe_pair`（loopback acceptor + TCP_NODELAY），服务端线程 echo 模式。

---

## LatencyBench — 延迟

测量单次操作的端到端延迟分布（p50/p90/p99/avg）。

| 测试项 | 关键指标 |
|--------|----------|
| 连接建立延迟 | p50/p90/p99 μs |
| 隧道转发延迟 (64B~64KB) | p50/p90/p99 μs |
| 小包延迟 (64B) | avg μs |
| 大包延迟 (64KB) | avg μs |

---

## MemoryBench — 内存管理

全面对比 PMR 内存分配器在不同资源下的性能。

| 测试项 | 关键指标 |
|--------|----------|
| 框架开销 (EmptyLoop, PauseResume) | ns（校准） |
| std::string vs memory::string 对比 | ns，多种大小 |
| PMR string：默认资源 / new_delete / 全局池 / 线程池 / 帧竞技场 | ns，多种大小 |
| 帧竞技场批量分配 + 重置 | ns |
| std::vector vs memory::vector 对比 | ns，8~4096 元素 |
| PMR vector：new_delete / 全局池 / 线程池 / 默认 / 帧竞技场 | ns |
| 多线程内存池竞争 (1/2/4/8 线程) | ops/s |
| 全局池 vs 线程池多线程对比 | ops/s |

---

## MuxBench — 多路复用帧编解码

测量 smux/yamux 帧序列化/反序列化和 UDP 数据报构建性能。

| 测试项 | 关键指标 |
|--------|----------|
| smux 帧序列化/反序列化 | MB/s |
| yamux 帧序列化/反序列化 | MB/s |
| smux UDP datagram 构建 (IPv4, 0~16KB) | MB/s |
| smux UDP length-prefixed 构建 (0~16KB) | MB/s |

---

## MuxConnBench — 多路复用连接吞吐量

测试帧编解码在不同 payload 大小和多流场景下的性能（不涉及 socket I/O）。

| 测试项 | 关键指标 |
|--------|----------|
| smux 帧序列化/反序列化 (16K/64K/128KB) | ns |
| yamux 帧序列化/反序列化 (16K/64K/128KB) | ns |
| smux 多流帧构建 (4/16/64 流) | μs |
| yamux 多流帧构建 (4/16/64 流) | μs |
| smux 128KB 帧内存分配统计 | ns |

---

## ProtocolBench — 代理协议端到端

测量各协议握手流程和数据处理的端到端性能。

| 测试项 | 关键指标 |
|--------|----------|
| HTTP 转发请求行构建 | ops/s |
| HTTP 大头部解析 (1K/4K/8KB) | MB/s |
| SOCKS5 UDP 头编码/解码、成功响应构建 | ops/s |
| Trojan UDP 包解析/构建 (0~4KB)、凭据验证 | ops/s |
| VLESS IPv4/域名请求解析、UDP 解析、响应生成 | ops/s |
| Shadowsocks 地址端口解析、PSK 解码 | ops/s |

---

## RealityBench — Reality TLS 伪装

对标 sing-box/Xray Reality 握手时间。

| 测试项 | 关键指标 |
|--------|----------|
| Reality 完整握手流程 | μs |
| 仅 X25519 密钥交换 | μs |
| 仅 TLS 伪装构建 | ns |
| 握手各阶段分解：密钥生成/公钥派生/交换/HKDF Extract/Expand | μs |
| Reality 握手内存占用 | ns |

---

## ResourceBench — 资源占用

测量内存分配速率和会话内存占用。

| 测试项 | 关键指标 |
|--------|----------|
| 内存分配速率：帧竞技场/线程池/全局池 (64KB) | ns |
| 会话内存占用 (100/1K 会话) | ms |
| PMR string 分配、vector push_back/resize | ns |

---

## ScaleBench — 连接扩展性

测量连接建立速率和多连接吞吐量扩展性。

| 测试项 | 关键指标 |
|--------|----------|
| 单连接建立时间 | μs |
| 连续建立 100 个连接速率 | ms |
| 单连接吞吐量 (16K/64K/128KB) | MB/s |
| 4 独立连接并发吞吐量 | MB/s |
| 8 独立连接并发吞吐量 | MB/s |

---

## SessionBench — 协议握手与数据传输

测量各协议握手时间和隧道数据传输吞吐量。

| 测试项 | 关键指标 |
|--------|----------|
| HTTP CONNECT 握手时间 | μs |
| SOCKS5 握手时间 | μs |
| Trojan 握手时间 | μs |
| VLESS 握手时间 | μs |
| SS2022 握手时间 | μs |
| 隧道数据传输吞吐量 (16K/64K/128KB) | MB/s |
| SS2022 AEAD 持续加解密吞吐量 (1K~64KB) | MB/s |

---

## 相关链接

- [[Testing]] — 测试体系
- [[dev/stress]] — 压力测试
- [[Configuration]] — 配置参考
- [[Crypto]] — 加密模块
- [[Memory]] — PMR 内存模块
- [[Multiplex]] — 多路复用模块
