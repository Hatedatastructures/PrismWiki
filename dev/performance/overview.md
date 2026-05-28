---
layer: dev
title: Prism 性能概览
tags: [dev, performance, overview]
---

# Prism 性能概览

Prism 代理服务器性能设计遵循 **零拷贝、零热路径 malloc、最大化硬件加速** 三大原则。本页概述性能架构与关键指标。

## 性能架构设计原则

### 1. 零拷贝传输

- 纯 TCP 透传使用 `std::span` 视图，无数据复制
- 大包场景吞吐达 **7.11 Gi/s**（接近内存带宽极限）
- 小包场景系统开销占主导，但已最小化协程切换

### 2. 热路径零 malloc

- [[pmr-memory-pool]] 提供三级内存池架构
- **Frame Arena**: 帧内临时对象，6-11 ns 分配，批量释放
- **ThreadLocal Pool**: 跨帧持久对象，17-75 ns，无竞争
- **Global Pool**: 全局共享数据，~120 ns（存在竞争开销）

### 3. 硬件加速最大化

- AES-NI 汇编指令启用（BoringSSL MinGW patch）
- AES-256-GCM 吞吐从 205 Mi/s 提升至 **13.7 Gi/s**（67x）
- X25519 密钥交换从 81.5 us 降至 **21.4 us**（3.8x）

## 核心性能指标

### 协议握手延迟

| 协议 | 延迟 | 说明 |
|:-----|:----:|:-----|
| SS2022 | 1.35 us | 纯内存，无网络往返 |
| Trojan | 131 us | 1 RTT |
| VLESS | 138 us | 1 RTT |
| SOCKS5 | 146 us | 2 RTT |
| HTTP | 170 us | 1 RTT |

### AEAD 加密吞吐

| 算法 | 吞吐 | AES-NI |
|:-----|:----:|:------:|
| AES-128-GCM | 16.2 Gi/s | 启用 |
| AES-256-GCM | 13.7 Gi/s | 启用 |

### 内存分配性能

| 分配器 | 耗时 | 适用场景 |
|:-------|:----:|:---------|
| Frame Arena | 6 ns | 帧内临时 |
| ThreadLocal | 17 ns | 跨帧持久 |
| Global Pool | 120 ns | 全局共享 |

## 性能子页面

- [[performance/tuning]] — 性能调优指南
- [[performance/profiling]] — 性能分析方法
- [[performance/report]] — 完整基准测试报告

## 相关架构

- [[ref/programming/pmr-concepts|pmr-memory-pool]] — PMR 内存池详解
- [[ref/programming/c++23-coroutines|cpp23-coroutines]] — 协程性能特性
- [[dev/debugging/tcp|tcp]] — TCP 传输优化
- [[dev/debugging/tls|tls]] — TLS 性能考量

## 参考资源

- [WireGuard Performance](https://www.wireguard.com/performance/)
- [Envoy Performance](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/performance)
- [TechEmpower Benchmarks](https://tfb-status.techempower.com/)