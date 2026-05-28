---
title: 性能问题排查指南
source:
  - I:/code/Prism/include/prism/
  - I:/code/Prism/src/prism/
  - I:/code/Prism/benchmarks/
module: debugging
type: reference
tags: [performance, latency, throughput, bottleneck, profiling, troubleshooting]
layer: dev
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/debugging/overview]]"
  - "[[dev/testing/stress|stress]]"
  - "[[dev/performance/report|performance/benchmark]]"
---

# 性能问题排查指南

性能问题包括延迟高、吞吐低、资源瓶颈等。本文档提供系统化的性能分析和优化方法。

## 概述

### 性能问题分类

| 问题类型 | 典型症状 | 影响范围 |
|----------|----------|----------|
| 延迟高 | 响应时间长 | 用户体验 |
| 吞吐低 | 处理能力不足 | 并发上限 |
| CPU 瓶颈 | CPU 使用率高 | 处理效率 |
| 内存瓶颈 | 内存压力大 | 稳定性 |
| IO 瓶颈 | 磁盘/网络阻塞 | 数据流转 |

### 性能分析维度

```
+--------------------------------------------------+
|                性能分析维度                        |
+--------------------------------------------------+
                       |
         +-------------+-------------+
         |             |             |
    +---------+   +---------+   +---------+
    | 延迟   |   | 吞吐   |   | 资源   |
    +---------+   +---------+   +---------+
         |             |             |
         v             v             v
    +---------+   +---------+   +---------+
    | RTT     |   | QPS     |   | CPU    |
    | 处理时间|   | BPS     |   | 内存   |
    | 等待时间|   | 连接数  |   | IO     |
    +---------+   +---------+   +---------+
```

---

## 详解

### 延迟分析

#### 延迟组成

```
总延迟 = 网络延迟 + 处理延迟 + 等待延迟

网络延迟:
- TCP 握手 (1-3 RTT)
- TLS 握手 (1-2 RTT)
- 数据传输

处理延迟:
- 协议解析
- 加密解密
- 内存分配

等待延迟:
- 队列等待
- 连接池等待
- 锁等待
```

#### 延迟测量方法

```bash
# TCP 连接延迟
tcptraceroute target_host

# HTTP 请求延迟
curl -w "time_total=%{time_total}\n" http://target/

# 使用 Prism 压测工具
# benchmarks/latency_test
```

#### 延迟优化点

| 阶段 | 优化方法 | 效果 |
|------|----------|------|
| TCP | TCP_FASTOPEN | 减少 1 RTT |
| TLS | TLS 1.3 | 1 RTT 握手 |
| TLS | Session Resumption | 0 RTT |
| 解析 | 预分配缓冲区 | 减少分配 |
| 加密 | AES-NI 硬件加速 | 提升速度 |

### 吞吐分析

#### 吞吐瓶颈定位

```
吞吐瓶颈
    |
    v
+------------------------+
| 1. 确定瓶颈位置         |
| - CPU 饱和？           |
| - 内存饱和？           |
| - IO 饱和？            |
| - 连接数饱和？         |
+------------------------+
    |
    v
+------------------------+
| 2. 分析瓶颈原因         |
| - 算法效率             |
| - 资源竞争             |
| - 配置限制             |
+------------------------+
    |
    v
+------------------------+
| 3. 制定优化方案         |
| - 算法优化             |
| - 并行化               |
| - 调整配置             |
+------------------------+
```

#### 吞吐测量方法

```bash
# 网络吞吐
iperf3 -c target_host -t 60

# HTTP 吞吐
wrk -t4 -c100 -d30s http://target/

# Prism 压测
# benchmarks/throughput_test
```

#### 吞吐优化配置

```json
{
    "pool": {
        "max_cache_per_endpoint": 256,    // 增加连接池
        "recv_buffer_size": 262144,       // 256KB 缓冲
        "send_buffer_size": 262144
    },
    "buffer": {
        "size": 524288                    // 512KB 传输缓冲
    }
}
```

### CPU 瓶颈分析

#### 症状特征

```bash
# CPU 使用率持续高
top -p $(pidof prism)
# CPU% > 80% 持续
```

#### CPU 分析工具

```bash
# perf 分析
perf top -p $(pidof prism)

# 热点分析
perf record -p $(pidof prism) -g -- sleep 60
perf report

# FlameGraph
perf script | stackcollapse-perf.pl | flamegraph.pl > cpu.svg
```

#### 常见 CPU 热点

| 热点 | 原因 | 解决方案 |
|------|------|----------|
| AEAD 加密 | 加密开销大 | AES-NI 硬件加速 |
| 内存分配 | 频繁分配 | PMR 池预分配 |
| 协议解析 | 字符串操作 | 避免拷贝，view |
| 日志输出 | 同步日志 | 异步日志队列 |

### 内存瓶颈分析

#### 症状特征

```bash
# 内存使用高
free -h
# Available 低
```

#### 内存分析

```bash
# 进程内存详情
pmap -x $(pidof prism)

# 内存统计
cat /proc/$(pidof prism)/status | grep Vm

# 内存分配统计
# 添加 debug 日志跟踪分配
```

#### 内存优化

| 方法 | 效果 | 适用场景 |
|------|------|----------|
| 减小缓冲区 | 减少占用 | 低并发 |
| 限制缓存 | 控制增长 | 长期运行 |
| Arena 重置 | 无碎片 | 高频分配 |

### IO 瓶颈分析

#### 症状特征

```bash
# 磁盘 IO 高
iostat -x 1
# %util 高

# 网络 IO 高
sar -n DEV 1
```

#### IO 分析

| IO 类型 | 分析工具 | 瓶颈指标 |
|---------|----------|----------|
| 网络读 | netstat -i | 丢包、错误 |
| 网络写 | ifconfig | 冲突、溢出 |
| 磁盘读 | iostat | await 高 |
| 磁盘写 | iostat | %util 高 |

---

## 日志分析示例

### 正常性能日志

```
[10:30:00.100] [info] Session started: latency=50ms
[10:30:05.100] [debug] Throughput: requests=100/s, bytes=10MB/s
[10:30:10.100] [debug] CPU usage: 30%, Memory: 512MB
```

### 性能瓶颈日志

```
[10:30:00.100] [warn] High latency detected: 500ms
[10:30:05.100] [warn] Throughput low: 10req/s, expected=100req/s
[10:30:10.100] [warn] CPU saturated: 95%
[10:30:15.100] [error] Connection pool exhausted
```

---

## 使用示例

### 性能监控脚本

```bash
#!/bin/bash
# monitor_performance.sh

PID=$(pidof prism)

echo "Performance monitoring for prism (PID: $PID)"

while true; do
    TIME=$(date '+%Y-%m-%d %H:%M:%S')
    
    # CPU
    CPU=$(ps -p $PID -o %cpu --no-headers)
    
    # Memory
    MEM=$(ps -p $PID -o rss --no-headers)
    
    # Connections
    CONN=$(ss -tn state established '( sport = :443 )' | wc -l)
    
    echo "$TIME: CPU=$CPU%, MEM=$MEMKB, CONN=$CONN"
    
    sleep 5
done
```

### 延迟测试脚本

```bash
#!/bin/bash
# test_latency.sh

TARGET=$1
PORT=$2

echo "Testing latency to $TARGET:$PORT"

for i in {1..10}; do
    # TCP 连接延迟
    START=$(date +%s%N)
    timeout 5 bash -c "echo > /dev/tcp/$TARGET/$PORT" 2>/dev/null
    END=$(date +%s%N)
    
    if [ $? -eq 0 ]; then
        LATENCY=$((($END - $START) / 1000000))
        echo "Test $i: $LATENCY ms"
    else
        echo "Test $i: timeout"
    fi
done
```

---

## 最佳实践

### 配置优化建议

| 参数 | 低延迟场景 | 高吞吐场景 |
|------|------------|------------|
| buffer.size | 64KB | 512KB |
| pool.max_cache | 32 | 256 |
| pool.connect_timeout | 100ms | 500ms |
| multiplex.initial_window | 64KB | 512KB |

### 性能监控建议

| 监控项 | 周期 | 阈值 |
|--------|------|------|
| CPU | 每 5s | > 80% 告警 |
| Memory | 每 30s | > 90% 告警 |
| Latency | 每 10s | > 100ms 告警 |
| Throughput | 每 30s | 下降 > 50% 告警 |

### 压测建议

```bash
# 渐进式压测
# 1. 低负载基准
wrk -t2 -c10 -d30s http://target/

# 2. 中等负载
wrk -t4 -c50 -d30s http://target/

# 3. 高负载
wrk -t8 -c200 -d60s http://target/

# 4. 极限测试
wrk -t16 -c500 -d120s http://target/
```

---

## 常见问题

### Q1: 延迟突然升高怎么办？

**A**: 
1. 检查网络状态
2. 检查服务器负载
3. 检查是否有 GC 或日志刷盘
4. 检查连接池是否等待

### Q2: 吞吐无法提升怎么办？

**A**: 
1. 定位瓶颈（CPU/内存/IO）
2. 分析热点代码
3. 调整并发配置
4. 考虑架构优化

### Q3: CPU 使用率异常高怎么办？

**A**: 
```bash
# 使用 perf 分析热点
perf top -p $(pidof prism)

# 查看是否是加密/日志/分配
```

### Q4: 内存不足怎么办？

**A**: 
1. 减小缓冲区配置
2. 减少连接池缓存
3. 检查是否有泄漏
4. 添加内存限制

---

## 排障指南

### 问题：响应延迟高

**症状**: 用户反馈响应慢

**排查步骤**:

1. 测量各阶段延迟
   ```bash
   # 网络延迟
   ping target_host
   
   # TCP 延迟
   tcptraceroute target_host
   ```

2. 检查服务器负载
   ```bash
   top -p $(pidof prism)
   ```

3. 检查队列等待
   ```bash
   # 查看日志中的等待记录
   grep "waiting" prism.log
   ```

---

### 问题：吞吐达不到预期

**症状**: 性能测试不达标

**排查步骤**:

1. 确认瓶颈
   ```bash
   # CPU 瓶颈？
   top -p $(pidof prism)
   
   # 内存瓶颈？
   free -h
   
   # 连接瓶颈？
   ss -tn | wc -l
   ```

2. 分析代码热点
   ```bash
   perf top -p $(pidof prism)
   ```

3. 检查配置限制
   ```json
   "pool": {
       "max_cache_per_endpoint": 64  // 是否太小？
   }
   ```

---

### 问题：CPU 100%

**症状**: CPU 使用率持续 100%

**排查步骤**:

1. 分析热点函数
   ```bash
   perf record -p $(pidof prism) -g -- sleep 30
   perf report
   ```

2. 检查是否是加密
   ```cpp
   // AEAD 加密是主要 CPU 消耗
   // 确认 AES-NI 是否启用
   ```

3. 检查是否是日志
   ```json
   "trace": {
       "log_level": "info"  // trace/debug 级别消耗大
   }
   ```

---

## 相关链接

- [[dev/debugging/overview]] — 排障体系概述
- [[dev/testing/stress|stress]] — 压力测试体系
- [[dev/performance/report|performance/benchmark]] — 性能基准测试
- [[dev/debugging/memory]] — 内存问题排查
- [[dev/debugging/connection]] — 连接问题排查