---
title: 压力测试
tags: [stress, memory, pool, arena, multiplex, concurrency]
source:
  - I:/code/Prism/stresses/MemoryStress.cpp
  - I:/code/Prism/stresses/PoolContentionStress.cpp
  - I:/code/Prism/stresses/ArenaOverflowStress.cpp
  - I:/code/Prism/stresses/MuxStress.cpp
---

# 压力测试

Prism 提供 4 个压力测试工具，验证内存管理和多路复用模块在极端负载下的稳定性。

## 构建与运行

```bash
# 构建（需启用 PRISM_ENABLE_STRESS=ON，默认已启用）
cmake -B build_release -DCMAKE_BUILD_TYPE=Release -DPRISM_ENABLE_STRESS=ON
cmake --build build_release --config Release

# 运行
build_release/stresses/MemoryStress.exe
build_release/stresses/PoolContentionStress.exe
build_release/stresses/ArenaOverflowStress.exe
build_release/stresses/MuxStress.exe
```

---

## MemoryStress — 内存池压力测试

**源码**: `stresses/MemoryStress.cpp`

测试 PMR 内存分配器在高并发、大容量分配/释放场景下的稳定性。

- **线程数**: 4（默认，等于 `std::thread::hardware_concurrency()` 或硬编码）
- **持续时间**: 10 秒
- **最大内存**: 2 GB（自动检测系统可用内存并取 85% 安全限制）
- **分配大小**: 64 ~ 65536 字节随机
- **每批对象数**: 1000

**验证点**:
- 多线程并发分配/释放不崩溃
- 无 OOM 错误（`std::bad_alloc`）
- 操作数 > 0
- 使用 `counting_resource` 统计分配/释放字节数和峰值内存

**预期行为**: 正常退出（return 0），输出吞吐量和内存统计。

---

## PoolContentionStress — 池竞争压力测试

**源码**: `stresses/PoolContentionStress.cpp`

专门测试全局内存池在极端锁竞争下的表现。所有线程共享同一个 `global_pool()`。

- **线程数**: `std::thread::hardware_concurrency()`
- **持续时间**: 5 秒
- **分配大小**: 固定 128 字节
- **每批操作**: 1000 次 `memory::string` 分配/释放

**验证点**:
- 吞吐量 ≥ 10000 ops/s（低于此阈值判定为 FAIL）
- 操作数 > 0

**预期行为**: 全局池的吞吐量应远高于线程池（因为线程本地池无锁竞争），此测试专门验证全局池在竞争下的性能底线。

---

## ArenaOverflowStress — 竞技场溢出压力测试

**源码**: `stresses/ArenaOverflowStress.cpp`

测试 `frame_arena` 在大量分配/重置循环下的延迟稳定性和吞吐量。

- **迭代次数**: 20000
- **每轮分配数**: 64 个对象
- **对象大小**: 1024 字节

**验证点**:
- 平均延迟 < 100 μs/batch（100000 ns）
- 批次数 > 0

**预期行为**: 帧竞技场的 reset 操作应保持稳定的低延迟，不因大量循环而退化。输出平均/最小/最大延迟和吞吐量。

---

## MuxStress — 多路复用压力测试

**源码**: `stresses/MuxStress.cpp`

测试 smux/yamux 帧编解码在高并发和大量数据下的稳定性。包含 4 个场景：

### 场景 1: 帧解码风暴（单线程）
- 100 万轮迭代
- 混合 smux（SYN/PSH/FIN/NOP）和 yamux（Data/WindowUpdate/Ping/GoAway）帧解码
- 每 10 轮穿插地址解析

### 场景 2: 并发编解码
- 4 线程，5 秒持续时间
- 每线程循环执行：smux 解码 + yamux 解码 + 地址解析 + UDP 构建/解析
- 使用 `counting_resource` 统计内存

### 场景 3: 地址解析覆盖
- 100 万轮迭代
- 覆盖 IPv4、短域名、长域名（200 字符）、IPv6 地址解析

### 场景 4: UDP 数据报往返验证
- 100 万轮迭代
- IPv4 datagram、域名 datagram、length-prefixed 三种格式的 build→parse 往返
- 验证解析结果与原始数据完全一致

**验证点**: 每个场景输出操作数、错误数和吞吐量。错误数应为 0。

---

## 相关链接

- [[Testing]] — 测试体系
- [[Benchmark]] — 基准测试
- [[Memory]] — PMR 内存模块
- [[Multiplex]] — 多路复用模块
