---
title: "ADR 005: 连接池设计"
created: 2026-05-23
updated: 2026-05-23
layer: dev
tags: [dev, adr, architecture, connection-pool, performance]
status: accepted
---

# ADR 005: 连接池设计

## 状态

已接受 (Accepted)

## 背景

代理服务器的核心工作模式是"接收客户端请求 -> 建立到目标服务器的连接 -> 双向转发数据"。每次代理请求都需要建立到目标服务器的 TCP 连接，涉及：

- **TCP 三次握手**: 1 个 RTT
- **TLS 握手**（如有）: 1-2 个 RTT
- **TCP 慢启动**: 初始窗口小，吞吐量低

对于频繁访问同一目标端点的场景（如访问同一 CDN 节点、同一 API 服务器），重复建立连接的开销显著。连接复用可以将握手开销均摊到多次请求上。

### 方案 A: 无连接池

每次请求新建连接，用完关闭。

- **优势**: 实现最简单，无状态管理
- **劣势**: 每次请求都承担握手开销；高频访问时浪费带宽和时间；目标服务器压力增大

### 方案 B: 全局连接池

所有 worker 线程共享一个连接池，通过锁保护并发访问。

- **优势**: 连接利用率最高，跨线程复用
- **劣势**: 锁竞争严重（全局互斥锁）；缓存行争用导致性能下降；不符合 Prism 的线程隔离设计

### 方案 C: 线程局部 LIFO 栈式连接池

每个 worker 线程维护独立的连接池，使用 LIFO（后进先出）栈式缓存。

- **优势**: 无锁设计（线程局部）；LIFO 保证最近使用的连接在栈顶（热连接优先复用）；实现简洁
- **劣势**: 跨线程无法复用连接；每个线程的连接池独立，总缓存连接数可能略多

## 决策

**选择方案 C: 线程局部 LIFO 栈式连接池**

### 核心数据结构

```cpp
class connection_pool {
    // 按端点组织的 LIFO 栈
    unordered_map<endpoint_key, vector<idle_item>, endpoint_hash> cache_;

    // 每个 idle_item 包含 socket 指针和最后使用时间
    struct idle_item {
        tcp::socket *socket;
        steady_clock::time_point last_used;
    };
};
```

### 连接获取流程

```
async_acquire(endpoint):
  1. 查找 cache_[endpoint_key(endpoint)]
  2. 如果栈非空:
     a. 弹出栈顶连接
     b. 过期检查: last_used + max_idle_seconds < now
     c. 健康检测: 非阻塞 peek 检查连接是否存活
     d. 通过 -> 返回 pooled_connection
     e. 失败 -> 关闭，继续弹出下一个
  3. 栈空或全部过期:
     a. co_spawn + timer 创建新连接
     b. 超时: connect_timeout_ms
     c. 设置 TCP_NODELAY、SO_KEEPALIVE
     d. 返回 pooled_connection
```

### 连接归还流程

`pooled_connection` 是 RAII 包装器，析构时自动调用 `recycle()`：

```
recycle(socket, endpoint):
  1. IPv6 过滤: cache_ipv6=false 时不缓存 IPv6 连接
  2. 健康检测: 非阻塞 peek
  3. 容量检查: 当前缓存数 < max_cache_per_endpoint
  4. 通过 -> 入栈等待复用
  5. 失败 -> 直接关闭 socket
```

### 后台清理

按 `cleanup_interval_sec`（默认 10 秒）周期性调用 `cleanup()`，遍历所有端点的连接栈，移除超过 `max_idle_seconds` 的过期连接。使用原地压缩算法，避免额外内存分配。

### RAII 连接包装器

```cpp
class pooled_connection {
    connection_pool *pool_;
    tcp::socket *socket_;
    tcp::endpoint endpoint_;

    ~pooled_connection();   // 自动归还或关闭
    auto release();         // 放弃所有权
    auto reset();           // 手动归还或关闭
};
```

### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_cache_per_endpoint` | 32 | 单端点最大缓存连接数 |
| `max_idle_seconds` | 30 | 空闲连接最大存活时间 |
| `connect_timeout_ms` | 300 | 新建连接超时 |
| `cleanup_interval_sec` | 10 | 后台清理间隔 |
| `recv_buffer_size` | 65536 | 接收缓冲区大小 |
| `send_buffer_size` | 65536 | 发送缓冲区大小 |
| `tcp_nodelay` | true | 启用 TCP_NODELAY |
| `keep_alive` | true | 启用 SO_KEEPALIVE |
| `cache_ipv6` | false | 是否缓存 IPv6 连接 |

### 运行统计

`pool_stats` 结构体提供运行时指标：

```cpp
struct pool_stats {
    size_t idle_count = 0;      // 当前空闲连接数
    size_t endpoint_count = 0;  // 有缓存的端点数
    size_t total_acquires = 0;  // 总获取次数
    size_t total_hits = 0;      // 缓存命中次数
    size_t total_creates = 0;   // 新建连接次数
    size_t total_recycles = 0;  // 归还次数
    size_t total_evictions = 0; // 驱逐次数
};
```

**关键指标**:
- **命中率** = `total_hits / total_acquires`，反映连接复用效率
- **驱逐率** = `total_evictions / total_recycles`，反映过期/不健康连接比例
- **空闲数** = `idle_count`，反映当前缓存水位

### 与全局池的性能对比

| 指标 | 全局锁保护池 | 线程局部 LIFO 池 |
|------|-------------|-----------------|
| 获取连接延迟 | 100-500ns（含锁） | 5-20ns（无锁） |
| 归还连接延迟 | 100-500ns（含锁） | 5-20ns（无锁） |
| 缓存行争用 | 高（全局计数器） | 无 |
| 跨线程复用 | 支持 | 不支持 |
| 总缓存连接数 | 较少（全局共享） | 较多（每线程独立） |

> 参考: [[core/connect/pool/pool|连接池模块]]

## 后果

### 优势

1. **连接复用率**:
   - LIFO 策略保证最近使用的连接在栈顶，这些连接的 TCP 拥塞窗口已经打开，吞吐量高
   - 对于高频访问同一端点的场景（CDN、API），连接复用率可达 80%+
   - 连接池统计（`pool_stats`）提供 `total_hits / total_acquires` 的命中率指标

2. **无锁设计**:
   - 连接池是线程局部的，`cache_` 的所有操作都在同一 `io_context` 线程中完成
   - 无需 `std::mutex` 或 `std::atomic`，分配/归还操作零同步开销
   - 符合 Prism 的线程隔离设计（每个 worker 独立 `io_context`）

3. **自动健康检测**:
   - 连接获取时执行非阻塞 peek 检测连接存活状态
   - 归还时再次检测，确保缓存中都是健康连接
   - 僵尸连接（对端已关闭但本地未感知）被自动清理

4. **RAII 资源管理**:
   - `pooled_connection` 的析构函数自动归还连接，避免资源泄漏
   - 支持 `release()` 放弃所有权（生命周期不受连接池管理）
   - 支持 `reset()` 手动归还

5. **可观测性**:
   - `pool_stats` 提供 idle_count、endpoint_count、acquires、hits、creates、recycles、evictions 等指标
   - 可用于监控连接池效率、诊断性能问题

### 劣势

1. **线程间无法复用**:
   - 线程 A 建立的连接只能在线程 A 复用，线程 B 无法使用
   - 当不同线程频繁访问相同端点时，总缓存连接数可能多于全局池方案
   - 但考虑到每个线程的 `io_context` 独立运行，连接无法跨线程使用（socket 与 `io_context` 绑定），这是架构约束而非设计缺陷

2. **LIFO 的局限性**:
   - LIFO 策略虽然热连接优先，但冷连接可能长期驻留在栈底
   - 依赖 `cleanup()` 定期清理过期连接，存在时间窗口内的无效连接
   - 如果 `cleanup_interval_sec` 设置过大，过期连接可能长时间占用资源

3. **容量控制**:
   - `max_cache_per_endpoint` 是硬上限，超出时直接关闭而不缓存
   - 对于端点数量多的场景，总缓存连接数 = 端点数 * max_cache_per_endpoint，可能占用大量文件描述符
   - 无全局总连接数限制

4. **健康检测的成本**:
   - 非阻塞 peek 虽然不阻塞，但涉及系统调用，在高频获取场景下有开销
   - 某些操作系统上 peek 可能返回假阳性（连接已半关闭但 peek 成功）

### 缓解措施

- 合理设置 `max_cache_per_endpoint`（默认 32），平衡复用率和资源占用
- `cleanup_interval_sec` 设为 `max_idle_seconds` 的 1/3，及时清理过期连接
- 通过 `pool_stats` 监控命中率，调整缓存参数
- `cache_ipv6 = false` 默认不缓存 IPv6 连接，减少无效缓存

## 相关页面

- [[core/connect/pool/pool|连接池模块]]
- [[dev/adr/001-pure-coroutine-architecture|ADR 001: 纯协程架构]]
- [[dev/adr/002-pmr-memory-strategy|ADR 002: PMR 内存策略]]
- [[dev/performance/overview|性能优化]]
- [[docs/configuration|配置说明]]
