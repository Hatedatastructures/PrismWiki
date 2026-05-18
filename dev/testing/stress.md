---
layer: dev
source: I:/code/Prism/stresses/
module: testing
type: reference
tags: [testing, stress, memory, pool, arena, multiplex, concurrency, performance]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing]]"
  - "[[dev/testing/overview]]"
  - "[[dev/testing/benchmark]]"
  - "[[dev/pmr-memory-pool]]"
  - "[[multiplex/bootstrap]]"
  - "[[crypto/hkdf]]"
---

# 压力测试体系

压力测试是验证系统在极端负载下稳定性和性能的关键手段。Prism 提供四个专业压力测试工具，覆盖内存管理、池竞争、竞技场溢出和多路复用四个核心维度。

## 概述

### 压力测试的目的

压力测试与基准测试（Benchmark）有本质区别：

| 测试类型 | 目标 | 关注指标 | 运行时间 |
|----------|------|----------|----------|
| 单元测试 | 功能正确性 | Pass/Fail | 毫秒级 |
| 基准测试 | 性能优化 | 吞吐量、延迟 | 秒级 |
| 压力测试 | 稳定性边界 | 无崩溃、无OOM、吞吐底线 | 分钟级 |
| 回归测试 | Bug修复验证 | 历史问题重现 | 秒级 |

压力测试的核心价值：

1. **边界探索**：找到系统在崩溃前的极限负载
2. **稳定性验证**：长时间运行不出现内存泄漏、死锁、竞态
3. **性能底线**：在极端竞争下确保最低吞吐量
4. **资源管理**：验证内存池、连接池、会话池的资源回收机制

### Prism 压力测试架构

```
                    +------------------------+
                    |    压力测试入口         |
                    +------------------------+
                              |
        +--------------------+--------------------+
        |                    |                    |
+---------------+  +------------------+  +------------------+
| MemoryStress |  | PoolContention   |  | ArenaOverflow    |
| 内存池极限    |  | 池竞争压力       |  | 竞技场溢出       |
+---------------+  +------------------+  +------------------+
        |
        +------------------+
        |
+------------------+
|    MuxStress    |
| 多路复用压力    |
+------------------+
        |
        +----+----+----+----+
        |    |    |    |
    场景1  场景2  场景3  场景4
    帧解码  并发编解码 地址解析 UDP往返
```

每个测试独立运行，互不干扰，输出详细的统计报告。

## 压力测试列表

| 压力测试 | 文件 | 测试目标 |
|----------|------|----------|
| MemoryStress | MemoryStress.cpp | 内存分配压力 |
| PoolContentionStress | PoolContentionStress.cpp | 内存池竞争 |
| ArenaOverflowStress | ArenaOverflowStress.cpp | 竞技场溢出处理 |
| MuxStress | MuxStress.cpp | 多路复用压力 |

## 构建与运行

### 构建配置

压力测试需要启用 `PRISM_ENABLE_STRESS=ON`（默认开启）：

```bash
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_STRESS=ON
cmake --build build_release --config Release

# 禁用压力测试
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_STRESS=OFF
```

### 运行全部压力测试

```bash
build_release/stresses/MemoryStress.exe
build_release/stresses/PoolContentionStress.exe
build_release/stresses/ArenaOverflowStress.exe
build_release/stresses/MuxStress.exe
```

### 一键运行完整套件

```bash
for test in MemoryStress PoolContentionStress ArenaOverflowStress MuxStress; do
    echo "=== Running $test ==="
    build_release/stresses/${test}.exe
    echo ""
done
```

预期输出格式：

```
[MemoryStress] Running...
  Threads: 4
  Duration: 10s
  Max Memory: 2GB
  ...
  Total operations: 12345678
  Peak memory: 1.5GB
  Status: PASS

[PoolContentionStress] Running...
  Throughput: 15000 ops/s
  Threshold: 10000 ops/s
  Status: PASS
```

---

## 详解

### MemoryStress — 内存池极限压力测试

**源码位置**: `stresses/MemoryStress.cpp`

MemoryStress 测试 PMR（Polymorphic Memory Resource）分配器在高并发、大容量分配/释放场景下的稳定性。

#### 测试参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 线程数 | 4 | 并发分配线程数 |
| 持续时间 | 10 秒 | 测试运行时间 |
| 最大内存 | 2 GB | 安全限制（自动检测系统可用内存并取 85%） |
| 分配大小 | 64 ~ 65536 字节 | 随机大小，模拟真实负载 |
| 每批对象数 | 1000 | 每轮分配的对象数量 |

#### 核心逻辑

```cpp
// MemoryStress 核心流程（简化）
void run_stress_test() {
    // 1. 初始化计数资源
    counting_resource counter;
    
    // 2. 创建工作线程
    std::vector<std::jthread> workers;
    for (int i = 0; i < thread_count; ++i) {
        workers.emplace_back([&counter, duration] {
            auto start = steady_clock::now();
            while (steady_clock::now() - start < duration) {
                // 3. 批量分配
                std::vector<void*> allocations;
                for (int j = 0; j < batch_size; ++j) {
                    size_t size = random_size();  // 64-65536
                    void* ptr = counter.allocate(size);
                    allocations.push_back(ptr);
                }
                
                // 4. 批量释放
                for (void* ptr : allocations) {
                    counter.deallocate(ptr);
                }
                
                // 5. 统计
                ops_count++;
            }
        });
    }
    
    // 6. 输出报告
    report(ops_count, counter.peak_memory());
}
```

#### 验证点

| 检查项 | 通过条件 | 失败判定 |
|--------|----------|----------|
| 多线程并发 | 无崩溃 | 段错误、死锁 |
| 内存分配 | 无 `std::bad_alloc` | OOM 错误 |
| 操作计数 | > 0 | 无任何分配 |
| 峰值内存 | < 系统限制 | 超过安全阈值 |

使用 `counting_resource` 统计总分配字节数、总释放字节数、峰值内存使用、分配次数。

#### 预期输出

```
[MemoryStress] Configuration:
  Threads: 4
  Duration: 10s
  Max Memory: 2048MB (85% of system)
  Allocation Size: 64-65536 bytes

[MemoryStress] Running...

[MemoryStress] Results:
  Total Allocations: 15678923
  Total Deallocations: 15678923
  Peak Memory: 1.2GB
  Throughput: 1.56M ops/s

[MemoryStress] Status: PASS
```

---

### PoolContentionStress — 池竞争压力测试

**源码位置**: `stresses/PoolContentionStress.cpp`

PoolContentionStress 专门测试全局内存池在极端锁竞争下的表现。所有线程共享同一个 `global_pool()`，最大化锁竞争。

#### 测试参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 线程数 | `hardware_concurrency()` | CPU 核心数 |
| 持续时间 | 5 秒 | 测试运行时间 |
| 分配大小 | 固定 128 字节 | 消除大小差异影响 |
| 每批操作 | 1000 次 | `memory::string` 分配/释放 |

#### 池竞争架构

```
+------------------+
|   Global Pool    |  <- 所有线程共享，有锁竞争
+------------------+
        |
        +----+----+----+
        |    |    |    |
+-----+ +-----+ +-----+
|TL-1 | |TL-2 | |TL-3 |  <- 线程本地池，无锁
+-----+ +-----+ +-----+
```

正常情况下线程优先使用本地池（无锁），但在本地池耗尽、跨线程释放、大对象分配时会访问全局池。本测试强制所有线程访问全局池，测试锁竞争下的性能底线。

#### 验证点

| 检查项 | 通过条件 | 失败判定 |
|--------|----------|----------|
| 吞吐量 | >= 10000 ops/s | < 10000 ops/s |
| 操作数 | > 0 | 无任何分配 |
| 稳定性 | 无死锁 | 程序挂起 |

#### 吞吐量阈值解读

10000 ops/s 是保守估计：

- 单次 `memory::string` 操作约 100-200ns（本地池）
- 全局池加锁后约 500-1000ns
- 4 核 CPU 下锁竞争更激烈，预计 2-5us
- 10000 ops/s = 每秒 10000 次，即每次 100us

实际吞吐量应远高于此（本地池可达百万级）。

---

### ArenaOverflowStress — 竞技场溢出压力测试

**源码位置**: `stresses/ArenaOverflowStress.cpp`

ArenaOverflowStress 测试 `frame_arena`（帧竞技场）在大量分配/重置循环下的延迟稳定性。

#### 帧竞技场工作原理

```cpp
void process_frame() {
    frame_arena arena;
    
    // 在帧内多次分配
    auto* data1 = arena.allocate(1024);
    auto* data2 = arena.allocate(2048);
    auto* data3 = arena.allocate(512);
    
    // 处理数据...
    
    // 帧结束，一次性释放所有内存
    arena.reset();  // O(1) 指针移动
}
```

优势：批量释放（一次 `reset()` 清理）、零碎片（连续分配）、低延迟（`reset()` 是 O(1) 操作）。

#### 测试参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 迭代次数 | 20000 | 分配/重置循环次数 |
| 每轮分配数 | 64 个对象 | 模拟帧内分配 |
| 对象大小 | 1024 字节 | 固定大小 |

#### 验证点

| 检查项 | 通过条件 | 失败判定 |
|--------|----------|----------|
| 平均延迟 | < 100 us/batch | >= 100 us |
| 批次数 | > 0 | 无任何批次完成 |

100us 是非常宽松的阈值。理论分配时间约 640ns，`reset()` 约 10ns，实际应 < 1us。超过阈值说明存在内存池碎片化、系统调用介入或内存对齐问题。

---

### MuxStress — 多路复用压力测试

**源码位置**: `stresses/MuxStress.cpp`

MuxStress 测试 smux/yamux 帧编解码在高并发和大量数据下的稳定性，包含四个场景。

#### 场景 1: 帧解码风暴（单线程）

测试纯解码吞吐量。

| 参数 | 值 |
|------|-----|
| 迭代次数 | 100 万轮 |
| 帧类型 | smux（SYN/PSH/FIN/NOP）+ yamux（Data/WindowUpdate/Ping/GoAway） |
| 地址解析 | 每 10 轮穿插一次 |

#### 场景 2: 并发编解码

测试多线程编解码稳定性。

| 参数 | 值 |
|------|-----|
| 线程数 | 4 |
| 持续时间 | 5 秒 |
| 操作 | smux 解码 + yamux 解码 + 地址解析 + UDP 构建/解析 |

使用 `counting_resource` 统计内存分配。

#### 场景 3: 地址解析覆盖

测试各种地址格式解析。

| 参数 | 值 |
|------|-----|
| 迭代次数 | 100 万轮 |
| 地址类型 | IPv4、短域名、长域名（200字符）、IPv6 |

```
"192.168.1.1:443"                          // IPv4
"example.com:443"                          // 短域名
"very-long-domain-name-200chars...com:443" // 长域名
"[::1]:443"                                // IPv6
```

#### 场景 4: UDP 数据报往返验证

测试 UDP 数据报的 build-parse 循环一致性。

| 参数 | 值 |
|------|-----|
| 迭代次数 | 100 万轮 |
| 格式 | IPv4 datagram、域名 datagram、length-prefixed |

#### 验证点汇总

| 场景 | 验证点 | 通过条件 |
|------|--------|----------|
| 场景 1 | 解码错误数 | = 0 |
| 场景 2 | 内存泄漏 | 分配 = 释放 |
| 场景 3 | 解析错误数 | = 0 |
| 场景 4 | 数据一致性 | 所有往返匹配 |

---

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

```
Time: 60 seconds
Connections: 1000 concurrent
Operations: 1000000 create/destroy cycles
```

### 场景 2: 大数据传输

```
Data size: 1GB per stream
Streams: 64 concurrent
Transfer mode: bidirectional
```

### 场景 3: 内存极限

```
Allocation: 90% of available memory
Duration: 30 minutes
Operation: continuous allocate/free
```

### 场景 4: 多路复用饱和

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

## 使用示例

### 自定义压力测试参数

```cpp
// 在 MemoryStress.cpp 中修改参数
constexpr int kThreadCount = 8;        // 增加线程数
constexpr int kDurationSec = 30;       // 延长测试时间
constexpr size_t kMaxMemoryGB = 4;     // 增大内存限制
```

```cpp
// 在 MuxStress.cpp 中修改迭代次数
constexpr int kIterations = 10_000_000;  // 10M 轮
```

### 嵌入 CI/CD 流程

```yaml
# GitHub Actions 示例
- name: Run Stress Tests
  run: |
    cmake -B build -DCMAKE_BUILD_TYPE=Release
    cmake --build build --config Release
    
    # 运行压力测试，超时 10 分钟
    timeout 600 build/stresses/MemoryStress.exe || exit 1
    timeout 600 build/stresses/PoolContentionStress.exe || exit 1
    timeout 600 build/stresses/ArenaOverflowStress.exe || exit 1
    timeout 600 build/stresses/MuxStress.exe || exit 1
```

### 性能回归检测

```bash
# 创建性能基线
cat > stress_baseline.txt << EOF
MemoryStress: 1500000 ops/s
PoolContentionStress: 20000 ops/s
ArenaOverflowStress: 1.0 us/batch
MuxStress: 0 errors all scenes
EOF

# 每次运行后对比
./run_stress.sh | tee current_result.txt
diff stress_baseline.txt current_result.txt
```

### 长时间运行建议

```bash
# 运行 10 分钟
build_release/stresses/MemoryStress.exe --duration=600

# 运行 30 分钟
build_release/stresses/MuxStress.exe --duration=1800

# 运行 1 小时（生产前验证）
build_release/stresses/MemoryStress.exe --duration=3600
build_release/stresses/MuxStress.exe --duration=3600
```

## 与基准测试对比

| 特性 | 压力测试 | 基准测试 |
|------|----------|----------|
| 目标 | 稳定性验证 | 性能测量 |
| 时间 | 长时间运行 | 短时间迭代 |
| 负载 | 极限负载 | 正常负载 |
| 输出 | 错误报告 | 时间统计 |
| 用途 | 生产验证 | 性能优化 |

## 最佳实践

### 压力测试运行时机

| 时机 | 原因 | 建议测试 |
|------|------|----------|
| 代码提交前 | 验证改动无性能退化 | 全部 |
| 每周 CI | 长期稳定性监控 | 全部 |
| 内存池修改后 | 验证分配器正确性 | MemoryStress + PoolContention |
| 多路复用修改后 | 验证帧处理正确性 | MuxStress |
| 性能优化后 | 验证优化效果 | 全部对比 |

### 资源配置建议

| 配置项 | 建议值 | 原因 |
|--------|--------|------|
| 系统内存 | >= 测试最大内存 * 2 | 避免 OOM 影响其他进程 |
| CPU 核心 | >= 4 | 充分测试并发竞争 |
| 运行时间 | >= 10s | 确保稳定状态 |
| 后台进程 | 最少 | 减少干扰 |

### 性能优化参考值

| 测试 | 基线值 | 优化目标 |
|------|--------|----------|
| MemoryStress | 1M ops/s | 2M ops/s |
| PoolContentionStress | 15K ops/s | 30K ops/s（使用线程本地池） |
| ArenaOverflowStress | 1us/batch | 0.5us/batch |
| MuxStress | 1M frames/s | 2M frames/s |

## 常见问题

### Q1: 压力测试运行很慢，是否正常？

**A**: 正常。压力测试设计为长时间运行（5-30秒）以暴露稳定性问题。如果运行时间过长（超过几分钟），检查系统内存不足、CPU 被其他进程占用、或死锁（程序挂起）。

### Q2: PoolContentionStress 吞吐量只有 5000 ops/s，如何优化？

**A**: 这说明全局池竞争严重。优化方向：确保线程本地池正确初始化、减少跨线程对象传递、增大本地池容量、使用无锁数据结构。

### Q3: MemoryStress 报告 std::bad_alloc，但系统内存充足？

**A**: 可能原因：地址空间碎片化（32 位系统）、单个分配超过限制、PMR 池耗尽（检查池容量配置）。

### Q4: MuxStress 场景 4 有解析错误？

**A**: 检查 UDP 数据报格式是否正确、地址编码是否规范、length-prefixed 格式的长度字段是否匹配。

### Q5: 压力测试可以并行运行吗？

**A**: 不建议。压力测试会最大化系统资源使用，并行运行会相互干扰，结果不可靠。建议顺序运行或在不同机器上运行。

### Q6: 如何在 CI 中自动判定压力测试结果？

**A**: 压力测试输出包含明确的 PASS/FAIL 状态。可以解析输出：

```bash
result=$(build/stresses/MemoryStress.exe | grep "Status:")
if [[ "$result" == *"PASS"* ]]; then
    echo "Test passed"
else
    echo "Test failed"
    exit 1
fi
```

## 排障指南

### 问题：MemoryStress 崩溃

**症状**: 程序异常退出，段错误

**排查步骤**:
1. 检查崩溃位置：使用调试器查看调用栈
2. 可能原因：多线程竞态条件、内存池实现错误、计数资源溢出
3. 解决方案：逐线程运行排查竞态、检查 memory::pool 实现、增加计数资源的范围检查

### 问题：PoolContentionStress 吞吐量过低

**症状**: 吞吐量 < 5000 ops/s

**排查步骤**:
1. 确认是否真的使用全局池
2. 检查锁实现（是否使用公平锁导致性能较低）
3. 解决方案：切换到非公平锁、增加线程本地池使用率、批量操作减少锁获取次数

### 问题：ArenaOverflowStress 延迟波动大

**症状**: 平均延迟正常，但最大延迟异常高

**排查步骤**:
1. 记录每次延迟分析分布
2. 检查是否有系统调用介入（内存池耗尽导致 `new` 调用）
3. 解决方案：增大竞技场初始容量、确保所有分配使用池、预热竞技场减少首次分配延迟

### 问题：MuxStress 解码错误

**症状**: 场景 1 或 2 有解码错误

**排查步骤**:
1. 定位错误帧，记录原始数据
2. 检查 smux/yamux 头部字段和长度字段是否匹配
3. 解决方案：检查帧构建代码、添加帧格式验证、更新协议实现

### 问题：压力测试导致系统卡顿

**症状**: 运行压力测试时系统响应变慢

**排查步骤**:
1. 检查 CPU 和内存占用
2. 降低测试参数、在专用测试机器上运行、设置 CPU/内存限制

## 相关链接

- [[dev/testing]] — 测试体系概览
- [[dev/testing/overview]] — 测试体系概述
- [[dev/testing/benchmark]] — 基准测试体系
- [[dev/pmr-memory-pool]] — PMR 内存池
- [[multiplex/bootstrap]] — 多路复用引导
- [[multiplex/duct]] — 多路复用管道
