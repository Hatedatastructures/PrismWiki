---
title: 内存问题排查指南
source:
  - I:/code/Prism/include/prism/memory/
  - I:/code/Prism/src/prism/memory/
module: debugging
type: reference
tags: [memory, pmr, leak, fragmentation, pool, troubleshooting]
layer: dev
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/debugging/overview]]"
  - "[[memory]]"
  - "[[dev/coding/pmr|pmr]]"
---

# 内存问题排查指南

Prism 使用 PMR（Polymorphic Memory Resource）三级内存池架构。本文档提供内存泄漏、碎片等问题的排查方法。

## 概述

### 内存问题分类

| 问题类型 | 典型症状 | 影响范围 |
|----------|----------|----------|
| 内存泄漏 | 内存持续增长 | 长期运行 |
| 内存碎片 | 分配失败但总量足够 | 高频分配 |
| 池溢出 | 超出池容量 | 高并发 |
| 僵尸对象 | 对象未释放 | 会话管理 |
| 性能下降 | 分配延迟高 | 内存压力 |

### PMR 三级架构

```
+--------------------------------------------------+
|                PMR 三级内存池架构                  |
+--------------------------------------------------+
                       |
         +-------------+-------------+
         |             |             |
    +---------+   +---------+   +---------+
    | 全局池  |   | 线程池  |   | 帧 Arena|
    | Global  |   | Thread  |   | Frame   |
    +---------+   +---------+   +---------+
         |             |             |
    同步分配       线程本地      帧内临时
    大块内存       快速分配      生命周期绑定
```

各级特点：

| 级别 | 特点 | 适用场景 |
|------|------|----------|
| Global Pool | 同步、大块、长期 | 全局资源 |
| Thread Pool | 线程本地、快速 | 常规分配 |
| Frame Arena | 帧内、零释放开销 | 临时对象 |

---

## 详解

### 内存泄漏排查

#### 症状特征

```
日志示例:
[2026-05-17 10:30:45.123] [prism] [warn] Memory usage high: allocated=2GB, limit=4GB
[2026-05-17 10:40:45.456] [prism] [warn] Memory growth detected: +500MB in 10min
```

系统层面：
```bash
# 内存持续增长
top -p $(pidof prism)
# RSS 持续上升不回落
```

#### 排查流程

```
内存泄漏
    |
    v
+------------------------+
| 1. 确认泄漏类型         |
| - 进程内存增长         |
| - 池内存增长           |
| - 对象计数增长         |
+------------------------+
    |
    v
+------------------------+
| 2. 定位泄漏来源         |
| - 使用内存分析工具     |
| - 检查对象生命周期     |
+------------------------+
    |
    v
+------------------------+
| 3. 检查代码路径         |
| - 异常处理是否释放     |
| - 协程是否正确结束     |
| - shared_ptr 循环引用  |
+------------------------+
    |
    v
+------------------------+
| 4. 修复和验证           |
| - 修复代码             |
| - 压力测试验证         |
+------------------------+
```

#### 内存分析工具

**Linux 平台**

```bash
# Valgrind 内存检测
valgrind --leak-check=full --show-leak-kinds=all ./prism

# AddressSanitizer（编译时启用）
cmake -DCMAKE_CXX_FLAGS="-fsanitize=address" ..
cmake --build build

# heaptrack（Linux）
heaptrack ./prism
heaptrack_print heaptrack.prism.*.gz
```

**Windows 平台**

```powershell
# Visual Studio 内存诊断
# Debug → Windows → Memory Usage

# Dr. Memory
drmemory.exe -light -no_check_uninitialized -batch prism.exe
```

#### 常见泄漏模式

| 模式 | 原因 | 解决方案 |
|------|------|----------|
| 异常跳过释放 | try-catch 未捕获资源 | RAII 封装 |
| 协程挂起 | 协程未正常结束 | 使用 `co_await` 完成路径 |
| 循环引用 | shared_ptr 双向引用 | weak_ptr 打破循环 |
| 全局缓存 | 无限增长 | 设置上限或 TTL |
| 会话残留 | 会话未正确关闭 | 确认 close 路径 |

### 内存碎片排查

#### 症状特征

```
日志示例:
[2026-05-17 10:35:00.123] [prism] [error] Allocation failed: size=64KB, available=100MB
[2026-05-17 10:35:00.456] [prism] [warn] Pool fragmentation: wasted=50MB
```

#### 碎片来源

| 来源 | 特征 | 解决方案 |
|------|------|----------|
| 不均匀分配 | 大小差异大 | 分级池 |
| 频繁分配释放 | 高频操作 | Arena 批量 |
| 长短生命周期混合 | 混合管理 | 分区管理 |

#### PMR 碎片控制

Prism 的 PMR 设计已考虑碎片：

```cpp
// 线程池使用 monotonic_buffer_resource
// 不回收单块，整块回收
std::pmr::monotonic_buffer_resource thread_pool;

// 帧 Arena 更简单
// reset() 整体重置，无碎片
frame_arena arena;
arena.reset();  // 全部释放
```

### 池溢出排查

#### 症状特征

```
日志示例:
[2026-05-17 10:40:00.123] [prism] [error] Pool overflow: pool=thread_local, capacity=64MB
[2026-05-17 10:40:00.456] [prism] [warn] Fallback to global pool
```

#### 配置调整

```cpp
// 调整池大小
constexpr size_t kThreadPoolCapacity = 128 * 1024 * 1024;  // 128MB
constexpr size_t kFrameArenaSize = 4 * 1024 * 1024;         // 4MB
```

### 僵尸对象排查

#### 症状特征

```
日志示例:
[2026-05-17 10:45:00.123] [prism] [warn] Zombie session detected: id=abc123, age=3600s
[2026-05-17 10:45:00.456] [prism] [warn] Session count abnormal: count=10000, expected=100
```

#### 排查流程

```
僵尸对象
    |
    v
+------------------------+
| 1. 检查对象生命周期     |
| - 创建点               |
| - 预期销毁点           |
| - 实际状态             |
+------------------------+
    |
    v
+------------------------+
| 2. 检查持有者           |
| - 谁持有引用           |
| - 引用计数             |
| - 持有时长             |
+------------------------+
    |
    v
+------------------------+
| 3. 检查关闭路径         |
| - close() 是否调用     |
| - 异常是否跳过         |
| - 协程是否结束         |
+------------------------+
```

---

## 日志分析示例

### 正常内存使用

```
[10:30:00.100] [debug] Memory pool initialized: type=thread_local, capacity=64MB
[10:30:00.200] [info] Session created: id=abc123, memory=4KB
[10:30:05.100] [debug] Memory usage: global=0MB, thread=10MB, arena=0.5MB
```

### 内存泄漏日志

```
[10:30:00.100] [info] Session created: id=abc123
[10:35:00.100] [warn] Memory growth: +100MB in 5min
[10:40:00.100] [warn] Memory usage high: 2GB
[10:45:00.100] [error] Memory allocation failed
```

### 僵尸会话日志

```
[10:30:00.100] [info] Session created: id=abc123
[10:40:00.100] [debug] Session still active: id=abc123, duration=10min
[10:50:00.100] [warn] Zombie session: id=abc123, no activity for 20min
[11:00:00.100] [warn] Session cleanup forced: id=abc123
```

---

## 使用示例

### 内存监控脚本

```bash
#!/bin/bash
# monitor_memory.sh

PID=$(pidof prism)

echo "Monitoring prism memory (PID: $PID)"

while true; do
    # RSS 内存使用
    RSS=$(ps -p $PID -o rss --no-headers)
    
    # 当前时间
    TIME=$(date '+%Y-%m-%d %H:%M:%S')
    
    echo "$TIME: RSS = ${RSS}KB"
    
    sleep 60
done
```

### 内存分析命令

```bash
# 查看进程内存映射
pmap -x $(pidof prism)

# 查看内存统计
cat /proc/$(pidof prism)/status | grep -E "VmRSS|VmSize|VmData"

# 查看内存峰值
cat /proc/$(pidof prism)/status | grep VmHWM
```

---

## 最佳实践

### 内存配置建议

| 场景 | Thread Pool | Frame Arena | 建议 |
|------|-------------|-------------|------|
| 高并发 | 128MB+ | 4MB+ | 大池，减少回退 |
| 低并发 | 64MB | 2MB | 小池，节省内存 |
| 内存敏感 | 32MB | 1MB | 最小配置 |

### RAII 使用建议

```cpp
// 使用 RAII 包装资源
class SessionGuard {
public:
    explicit SessionGuard(Session* s) : session_(s) {}
    ~SessionGuard() {
        if (session_) session_->close();
    }
private:
    Session* session_;
};

// 使用时
void process() {
    SessionGuard guard(session);
    // ... 处理逻辑
    // 自动释放，即使异常
}
```

### Arena 使用建议

```cpp
// 帧内临时分配
void handle_frame() {
    frame_arena arena;
    
    // 多次分配
    auto* buf1 = arena.allocate(1024);
    auto* buf2 = arena.allocate(2048);
    
    // 处理完毕，整体重置
    arena.reset();  // 无碎片，高效
}
```

---

## 常见问题

### Q1: 如何确认是否有内存泄漏？

**A**: 
```bash
# 观察内存增长
top -p $(pidof prism)

# 运行 1 小时，内存持续增长则可能泄漏
# 正常情况应波动但稳定
```

### Q2: Valgrind 太慢怎么办？

**A**: 
- 使用 AddressSanitizer（编译选项）
- 减少 leak-check 范围
- 只检测特定代码路径

### Q3: 内存碎片怎么处理？

**A**: 
- 使用 PMR monotonic_buffer_resource
- 批量分配后整体释放
- 避免大小差异大的混合分配

### Q4: Arena 内存不释放怎么办？

**A**: 
- 确认 `reset()` 是否调用
- 检查帧处理是否完成
- 确认生命周期绑定正确

---

## 排障指南

### 问题：进程内存持续增长

**症状**: RSS 持续上升，不回落

**排查步骤**:

1. 使用 Valgrind 检测
   ```bash
   valgrind --leak-check=full ./prism
   ```

2. 检查对象计数
   ```bash
   # 查看日志中的 session count
   grep "Session count" prism.log
   ```

3. 检查全局缓存
   ```cpp
   // 查找无限增长的全局对象
   grep -r "static.*vector" include/
   grep -r "static.*map" include/
   ```

---

### 问题：分配失败但内存足够

**症状**: 内存总量足够但分配失败

**排查步骤**:

1. 检查碎片程度
   ```bash
   pmap -x $(pidof prism) | tail -20
   ```

2. 检查池配置
   ```cpp
   // 查看池大小
   constexpr size_t kThreadPoolCapacity = ...;
   ```

3. 检查分配大小
   ```cpp
   // 是否尝试分配超大块
   ```

---

### 问题：会话对象不释放

**症状**: Session 计数不减少

**排查步骤**:

1. 检查会话日志
   ```bash
   grep "Session created" prism.log | wc -l
   grep "Session closed" prism.log | wc -l
   ```

2. 检查持有者
   ```cpp
   // Session 使用 shared_ptr
   // 检查谁持有引用
   ```

3. 检查关闭路径
   ```cpp
   // 是否有 close() 未调用
   // 异常是否跳过清理
   ```

---

## 相关链接

- [[dev/debugging/overview]] — 排障体系概述
- [[memory]] — Memory 内存模块
- [[dev/coding/pmr|pmr]] — PMR 内存池详解
- [[core/instance/session/session|session]] — Session 会话管理