---
layer: dev
---

# Prism 性能调优指南

本页提供 Prism 代理服务器的性能调优方法，涵盖内存管理、加密优化、网络传输三个核心维度。

## 内存管理优化

### 选择正确的分配器

Prism 提供三级 PMR 内存池，根据场景选择：

| 场景 | 推荐分配器 | 耗时 | 特点 |
|:-----|:-----------|:-----|:-----|
| 帧内临时对象 | Frame Arena | 6-11 ns | 无竞争，批量释放 |
| 跨帧持久对象 | ThreadLocal Pool | 17-75 ns | 多线程友好 |
| 全局共享数据 | Global Pool | ~120 ns | 需接受竞争开销 |
| 大块数据 (>64KB) | NewDelete | ~25 ns | 避免 pool fragmentation |

### 代码示例

```cpp
// 帧内临时对象使用 Frame Arena
void process_frame() {
    memory::string temp_buffer{memory::frame_arena()};  // 6 ns 分配
    // ... 处理 ...
    // 函数结束自动释放（无需手动 free）
}

// 跨帧持久对象使用 ThreadLocal
class connection {
    memory::vector<uint8_t> buffer_{memory::thread_local_pool()};
};
```

### 多线程竞争优化

当前 **Global Pool 在 4 线程下退化 35x**（142 ns → 3530 ns）。

**解决方案**: 使用分片池 (`sharded_pool`)，每个线程独占分片：

```cpp
class sharded_pool : public std::pmr::memory_resource {
    static constexpr std::size_t shard_count = 8;
    struct alignas(64) aligned_shard {  // cache line 对齐
        std::pmr::unsynchronized_pool_resource pool;
    };
    std::array<aligned_shard, shard_count> shards_;
    // ...
};
```

预期效果: 4T 下 3530 ns → <150 ns

## 加密性能优化

### AES-NI 硬件加速

确保 AES-NI 汇编指令已启用：

**验证方法**:
```bash
# 运行加密基准
build_release/benchmarks/CryptoBench.exe

# 检查 AES-256-GCM 吞吐是否 >10 Gi/s
# 若 <500 Mi/s，说明 AES-NI 未启用
```

**MinGW 特殊处理**:
BoringSSL 的 Go 汇编生成器在 MinGW 下可能不兼容。解决方案：
1. 应用 MinGW compatibility patch
2. 或使用 MSVC 编译
3. 或 fallback 到 OpenSSL

### 密钥派生优化

| 操作 | 当前性能 | 优化建议 |
|:-----|:--------:|:---------|
| BLAKE3 | 185 Mi/s | 足够，无需优化 |
| HKDF Extract | 202.5 Mi/s | 足够 |
| HKDF Expand | 158.5 Mi/s | 足够 |

## 网络传输优化

### TCP 小包优化

当前 64B TCP Echo 延迟 20.1 us，系统开销占主导。

**优化方案**:
1. 合并读写：读取到缓冲区后一次性写入，减少协程挂起
2. 检查 `reliable.hpp` 中无多余包装层
3. Windows 下 ~5 us 理论极限需内核旁路

### 连接延迟优化

当前 P50=95 us，P99=336 us（3.5x 差距）。

**优化方案**:
1. 快速路径跳过健康检查（最近 1 秒内使用的连接）
2. 预创建机制：池空时后台预创建第二个连接
3. Happy Eyeballs 竞态取消优化

```cpp
// 快速路径示例
auto const now = std::chrono::steady_clock::now();
if ((now - item.last_used) < std::chrono::seconds(1)) {
    co_return item.conn;  // 跳过 healthy_fast()
}
```

### Happy Eyeballs 配置

```cpp
struct eyeball_config {
    std::chrono::milliseconds delay{250};      // 首端点失败后延迟
    std::chrono::milliseconds timeout{10000};  // 总超时
    bool prefer_ipv6{false};                   // IPv6 优先
};
```

## 性能回归监控

### 基准阈值

| 指标 | 基准函数 | 阈值 |
|:-----|:---------|:-----|
| AES-256-GCM | `BM_AeadContinuousSealAes256Gcm` | >500 Mi/s |
| X25519 | `BM_X25519KeyExchange` | <50 us |
| 全局池 4T | `BM_GlobalPoolConcurrent` | <150 ns |
| 连接 P99 | `BM_ConnectionLatencyP99` | <250 us |
| TCP 64B | `BM_TcpEcho64B` | <15 us |

### 运行回归检查

```bash
# 运行回归基准
build_release/benchmarks/RegressionBench.exe --json

# 对比 baselines.json
# 退化 >10% 会退出非零
```

## 相关页面

- [[performance/overview]] — 性能架构概览
- [[performance/profiling]] — 性能分析方法
- [[performance/report]] — 完整基准报告
- [[pmr-memory-pool]] — PMR 内存池详解

## 参考资源

- [Google Benchmark](https://github.com/google/benchmark)
- [Redis Benchmarks](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)