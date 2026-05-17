---
layer: dev
source: I:/code/Prism/stresses/
module: testing
type: reference
tags: [testing, stress, memory, concurrency, performance]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing/overview]]"
  - "[[dev/testing/benchmark]]"
  - "[[dev/stress]]"
---

# 压力测试体系

Prism 包含 4 个压力测试工具，用于验证系统在高负载下的稳定性。

## 压力测试列表

| 压力测试 | 文件 | 测试目标 |
|----------|------|----------|
| MemoryStress | MemoryStress.cpp | 内存分配压力 |
| PoolContentionStress | PoolContentionStress.cpp | 内存池竞争 |
| ArenaOverflowStress | ArenaOverflowStress.cpp | 竞技场溢出处理 |
| MuxStress | MuxStress.cpp | 多路复用压力 |

## 构建配置

压力测试需要启用 `PRISM_ENABLE_STRESS=ON`（默认开启）：

```bash
# 构建包含压力测试
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release

# 禁用压力测试
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_STRESS=OFF
```

## 运行压力测试

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

## MemoryStress — 内存分配压力

测试大规模内存分配和释放的稳定性。

### 测试内容

```
=== Memory Stress Test ===
Phase 1: Rapid allocate/free cycles
  - 1000000 allocations of 1KB
  - 1000000 allocations of 4KB
  - 1000000 allocations of 16KB
Phase 2: Sustained high allocation
  - 10000 concurrent allocations
Phase 3: Memory fragmentation test
  - Random size allocations
  - Interleaved allocate/free
Results:
  - Total allocations: 3000000+
  - Peak memory usage: XX MB
  - No memory leaks detected
  - Allocation time: stable throughout
```

### 测试目标

- 验证内存池在高分配频率下的稳定性
- 检测内存泄漏
- 测试内存碎片化影响
- 确保分配时间稳定

### 关键验证点

| 验证点 | 期望 |
|--------|------|
| 内存泄漏 | 无泄漏 |
| 分配时间 | 稳定，不随时间增长 |
| 碎片化 | 影响可控 |
| 峰值内存 | 符合预期 |

## PoolContentionStress — 内存池竞争

测试多线程环境下内存池的竞争处理。

### 测试内容

```
=== Pool Contention Stress Test ===
Thread count: 8
Iterations per thread: 1000000
Operations:
  - Thread-local pool allocate/free
  - Global pool allocate/free
  - Mixed pool operations
Results:
  - Total operations: 8000000
  - Contention events: XX
  - Average latency per operation: XX ns
  - Max latency: XX ns
  - No deadlocks detected
  - No data corruption
```

### 测试目标

- 验证线程安全
- 测试竞争条件处理
- 检测死锁风险
- 测量竞争延迟影响

### 关键验证点

| 验证点 | 期望 |
|--------|------|
| 死锁 | 无死锁 |
| 数据损坏 | 无损坏 |
| 竞争延迟 | 可控范围 |
| 线程公平性 | 良好 |

## ArenaOverflowStress — 竞技场溢出

测试帧竞技场在溢出情况下的处理。

### 测试内容

```
=== Arena Overflow Stress Test ===
Arena size: 64KB
Test phases:
  - Normal allocation (within limit)
  - Near-limit allocation
  - Overflow attempts
  - Reset and reuse
Results:
  - Overflow handling: correct
  - Fallback allocation: working
  - Arena reset: successful
  - No memory corruption
  - Performance degradation: minimal
```

### 测试目标

- 验证溢出处理机制
- 测试备用分配路径
- 确保竞技场重置正确
- 检测内存损坏

### 关键验证点

| 验证点 | 期望 |
|--------|------|
| 溢出检测 | 正确检测 |
| 备用分配 | 成功分配 |
| 重置后可用 | 可重新使用 |
| 内存安全 | 无损坏 |

## MuxStress — 多路复用压力

测试多路复用在高负载下的稳定性。

### 测试内容

```
=== Mux Stress Test ===
Configuration:
  - Connections: 4
  - Streams per connection: 64
  - Total streams: 256
Test phases:
  - Stream creation burst
  - Concurrent data transfer
  - Stream close burst
  - Connection recovery
Results:
  - Stream creation success: 100%
  - Data integrity: verified
  - Stream close: clean
  - Connection recovery: working
  - Frame processing: no errors
```

### 测试目标

- 验证大量流的创建和管理
- 测试并发数据传输
- 验证流关闭的清洁性
- 测试连接恢复机制

### 关键验证点

| 验证点 | 期望 |
|--------|------|
| 流创建 | 100% 成功 |
| 数据完整性 | 无损坏 |
| 流关闭 | 清洁退出 |
| 连接恢复 | 正确恢复 |

## 辅助工具

### CountingResource

`stresses/CountingResource.hpp` 提供资源计数跟踪：

```cpp
/**
 * @brief 计数资源跟踪器
 * @details 跟踪 PMR 资源的分配和释放，用于检测泄漏
 */
class CountingResource : public std::pmr::memory_resource {
public:
    size_t allocated_bytes() const;
    size_t allocation_count() const;
    size_t deallocation_count() const;
    bool has_leaks() const;
};
```

使用示例：

```cpp
CountingResource counter;
std::pmr::vector<int> vec(&counter);

// 进行分配操作...

if (counter.has_leaks()) {
    std::cout << "Memory leak detected!\n";
}
```

## 压力测试场景

### 场景 1: 高频连接创建

模拟高频连接创建/销毁：

```
Time: 60 seconds
Connections: 1000 concurrent
Operations: 1000000 create/destroy cycles
```

### 场景 2: 大数据传输

模拟大数据量传输：

```
Data size: 1GB per stream
Streams: 64 concurrent
Transfer mode: bidirectional
```

### 场景 3: 内存极限

模拟内存使用极限：

```
Allocation: 90% of available memory
Duration: 30 minutes
Operation: continuous allocate/free
```

### 场景 4: 多路复用饱和

模拟多路复用极限：

```
Connections: 4
Streams: 256 (max)
Operations: burst create + data + burst close
```

## 结果解读

### 正常结果

```
=== Stress Test PASSED ===
Duration: 60 seconds
Operations: 10000000
Errors: 0
Memory leaks: 0
Peak memory: 512 MB (expected)
Latency P99: 150 us (acceptable)
```

### 异常结果

```
=== Stress Test FAILED ===
Duration: 60 seconds
Operations: 10000000
Errors: 15
  - Error 1: Memory allocation timeout
  - Error 2: Stream creation failed (connection limit)
Memory leaks: 3 detected
Peak memory: 2 GB (unexpected high)
Latency P99: 5000 us (excessive)
```

## 与基准测试对比

| 特性 | 压力测试 | 基准测试 |
|------|----------|----------|
| 目标 | 稳定性验证 | 性能测量 |
| 时间 | 长时间运行 | 短时间迭代 |
| 负载 | 极限负载 | 正常负载 |
| 输出 | 错误报告 | 时间统计 |
| 用途 | 生产验证 | 性能优化 |

## 长时间运行建议

```bash
# 运行 10 分钟
build_release/stresses/MemoryStress.exe --duration=600

# 运行 30 分钟
build_release/stresses/MuxStress.exe --duration=1800

# 运行 1 小时（生产前验证）
build_release/stresses/MemoryStress.exe --duration=3600
build_release/stresses/MuxStress.exe --duration=3600
```

## 相关链接

- [[dev/testing/overview]] — 测试体系概述
- [[dev/testing/benchmark]] — 基准测试体系
- [[dev/stress]] — 压力测试总览
- [[dev/testing/concurrency]] — 并发测试
- [[dev/pmr-memory-pool]] — PMR 内存池