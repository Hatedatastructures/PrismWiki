---
title: "连接池"
category: "network"
type: ref
module: ref
source: "概念文档, 实践经验"
tags: [网络, 连接池, 性能, 复用, 缓存, 资源管理]
created: 2026-05-15
updated: 2026-05-17
---

# 连接池

**类别**: 网络

## 概述

连接池（Connection Pool）是一种用于管理和复用网络连接的资源管理技术，通过缓存已建立的连接避免频繁创建和销毁连接带来的开销。在代理服务器、数据库访问、HTTP 客户端等高并发场景中，连接池是性能优化的关键技术之一。

网络连接的建立和销毁是昂贵的操作。以 TCP 连接为例，建立连接需要三次握手，至少消耗一个 RTT（Round-Trip Time），在 WAN 环境下可能需要数百毫秒。SSL/TLS 握手更加昂贵，需要多个 RTT 和大量 CPU 计算。销毁连接需要四次挥手，主动关闭方需要等待 2MSL 的 TIME_WAIT 状态。频繁创建销毁连接不仅影响性能，还会消耗大量系统资源（端口、内存、CPU）。

连接池的核心思想是"复用"：连接使用完毕后不立即销毁，而是归还到池中等待下次使用。这消除了连接建立的开销，显著提高了性能。同时，连接池还提供资源管理功能，限制连接数量、监控连接状态、自动清理无效连接，防止资源泄漏和过度消耗。

连接池的核心价值：

**性能提升**：复用连接消除连接建立开销。对于 TLS 连接，避免了昂贵的握手过程，性能提升尤其明显。测试数据显示，连接池可将请求延迟降低 50%-90%。

**资源管理**：连接池限制最大连接数，防止资源过度消耗。同时统一管理连接生命周期，避免连接泄漏。连接池还提供健康检查，自动清理无效连接。

**流量控制**：通过连接数量限制实现并发控制，防止系统被过载请求淹没。连接池可以配合限流策略，平滑处理流量峰值。

**统计监控**：连接池提供详细的统计信息，包括活跃连接数、空闲连接数、命中率、等待时间等，便于监控和优化。

**负载均衡**：高级连接池可以根据负载分布连接，实现负载均衡。例如轮询、最少活跃连接、一致性哈希等策略。

### 适用场景

连接池适用于以下场景：

**高并发请求**：大量并发请求需要快速处理，连接建立开销成为瓶颈。代理服务器是典型场景。

**长连接服务**：服务端支持持久连接，连接可以长时间保持有效。HTTP/1.1 Keep-Alive、数据库连接是典型例子。

**SSL/TLS 连接**：加密连接握手开销大，复用连接显著提升性能。HTTPS 代理、安全数据库连接是典型场景。

**有限资源环境**：系统资源有限（端口、内存），需要精细管理连接数量。嵌入式系统、容器化环境是典型场景。

### 连接池类型

连接池可以按不同维度分类：

**按连接类型分类**：

| 类型 | 说明 | 典型应用 |
|------|------|---------|
| TCP 连接池 | 复用 TCP 连接 | HTTP 客户端、代理服务器 |
| SSL/TLS 连接池 | 复用加密连接 | HTTPS 客户端、安全代理 |
| 数据库连接池 | 复用数据库连接 | ORM、数据访问层 |
| Redis 连接池 | 复用 Redis 连接 | 缓存访问、消息队列 |

**按架构分类**：

| 类型 | 说明 | 特点 |
|------|------|------|
| 同步连接池 | 同步获取归还连接 | 简单，适合阻塞 I/O |
| 异步连接池 | 异步获取归还连接 | 高效，适合非阻塞 I/O |
| 协程连接池 | 协程友好接口 | 无阻塞，适合协程框架 |
| 线程安全连接池 | 支持多线程访问 | 使用锁或原子操作 |

**按策略分类**：

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| 固定大小池 | 固定连接数量 | 资源受限环境 |
| 动态大小池 | 按需增减连接数 | 流量波动场景 |
| 分片连接池 | 多池分片减少竞争 | 高并发场景 |
| 按端点池 | 每端点独立池 | 多目标场景 |

## 原理详解

### 连接池架构

典型连接池架构包含以下组件：

**连接存储**：
存储可用连接的数据结构，通常是队列或栈：
- FIFO 队列：先进先出，确保连接均匀使用
- LIFO 栈：后进先出，热点连接保持在栈顶，提高命中率
- 优先队列：按优先级排序，优先使用特定连接

**连接状态管理**：
跟踪连接状态的机制：
- IDLE：空闲，可用
- ACTIVE：活跃，正在使用
- VALIDATING：健康检查中
- INVALID：无效，待清理

**获取机制**：
从池中获取连接的策略：
- 立即获取：有空闲连接立即返回
- 创建新连接：池不满时创建新连接
- 等待：池满时等待连接归还
- 失败：超时返回错误

**归还机制**：
连接使用完毕归还的策略：
- 直接归还：放回池中
- 先验证：健康检查后归还
- 销毁：验证失败则销毁

**清理机制**：
清理无效连接的策略：
- 空闲超时清理：空闲超过阈值清理
- 最大生命周期：连接存在超过阈值清理
- 健康检查：定期检查连接有效性

**架构图**：
```
                    +----------------------------------+
                    |          Connection Pool         |
                    +----------------------------------+
                              |              |
              +---------------+              +---------------+
              |                               |
    +---------v---------+           +---------v---------+
    |   Idle Queue      |           |   Active Tracker  |
    |   [conn1, conn2]  |           |   {conn -> user}  |
    +-------------------+           +-------------------+
              |                               |
              |    acquire()                  |    release()
              +---------------+---------------+
                              |
                    +---------v---------+
                    |    Statistics     |
                    |   active/idle/    |
                    |   hit_rate/wait   |
                    +-------------------+
                              |
                    +---------v---------+
                    |   Cleaner         |
                    |   timeout/health  |
                    |   check           |
                    +-------------------+
```

### 工作流程

连接池的基本工作流程：

**获取连接（acquire）**：
```
1. 检查空闲队列
   ├── 有空闲连接 → 验证连接有效性 → 返回连接
   │   └── 无效 → 销毁连接 → 重新检查空闲队列
   └── 无空闲连接
       ├── 当前连接数 < 最大值 → 创建新连接 → 返回连接
       └── 当前连接数 >= 最大值
           ├── 有等待队列 → 等待连接归还
           └── 无等待队列/超时 → 返回错误
```

**归还连接（release）**：
```
1. 检查连接有效性
   ├── 有效 → 放入空闲队列
   │   ├── 有等待者 → 通知等待者
   │   └── 无等待者 → 连接变为空闲
   └── 无效 → 销毁连接
       └── 有等待者 → 创建新连接或唤醒其他等待者
```

**清理连接（cleanup）**：
```
1. 扫描空闲队列
   ├── 检查空闲时间 > 超时阈值 → 销毁连接
   ├── 检查生命周期 > 最大值 → 销毁连接
   └── 检查连接有效性 → 无效则销毁
2. 维护最小连接数
   ├── 连接数 < 最小值 → 创建新连接
   └── 保持池健康
```

### 配置参数详解

连接池的典型配置参数：

**容量参数**：

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| max_connections | 最大连接数 | CPU核心数 * 2 ~ CPU核心数 * 4 |
| min_connections | 最小连接数（预热） | max_connections / 4 |
| max_idle_connections | 最大空闲连接数 | max_connections / 2 |

**超时参数**：

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| max_idle_seconds | 空闲连接超时 | 60-300 秒 |
| max_lifetime_seconds | 连接最大生命周期 | 1800-3600 秒 |
| acquire_timeout_ms | 获取连接超时 | 100-5000 毫秒 |
| connect_timeout_ms | 新建连接超时 | 3000-10000 毫秒 |

**验证参数**：

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| validation_query | 验证查询（数据库） | SELECT 1 / ping |
| validation_interval_seconds | 验证间隔 | 30-60 秒 |
| validate_on_acquire | 获取时验证 | false（高性能） |
| validate_on_release | 归还时验证 | true（安全） |

### 按端点分池策略

当代理服务器连接多个不同目标时，通常采用按端点分池策略：

**端点标识**：
端点通常由以下组成：
- 目标主机名或 IP
- 目标端口
- 协议类型（TCP/TLS）
- 可选的连接属性

**端点池管理**：
```
Endpoint Pool Registry
+----------------------------------+
|  endpoint1.example.com:443 TLS   | → Pool1: [conn1, conn2, conn3]
|  endpoint2.example.com:80  TCP   | → Pool2: [conn4, conn5]
|  endpoint3.example.com:443 TLS   | → Pool3: [conn6]
+----------------------------------+
```

**优势**：
- 不同目标连接隔离管理
- 避免连接混用导致问题
- 可针对不同目标设置不同参数

**挑战**：
- 端点数量可能很大，内存占用高
- 不活跃端点的池需要清理
- 需要合理的上限和清理策略

### 异步与协程连接池

现代高性能网络应用通常使用异步或协程 I/O，连接池需要相应适配：

**异步连接池特点**：
- 获取连接不阻塞，返回 future/promise
- 支持回调式通知
- 适合事件驱动架构

**协程连接池特点**：
- 获取连接使用 await 语法
- 自动调度等待者
- 代码简洁直观

**协程连接池实现示例**：
```cpp
class coroutine_connection_pool {
public:
    // 获取连接
    auto acquire(std::string_view endpoint)
        -> net::awaitable<tcp::socket> {

        // 尝试从空闲队列获取
        auto conn = try_acquire_idle(endpoint);
        if (conn) {
            co_return std::move(*conn);
        }

        // 池未满，创建新连接
        if (can_create_new(endpoint)) {
            auto new_conn = co_await create_connection(endpoint);
            track_active(endpoint, new_conn);
            co_return std::move(new_conn);
        }

        // 池满，等待连接归还
        co_await wait_for_available(endpoint);

        // 获取归还的连接
        auto available = try_acquire_idle(endpoint);
        if (available) {
            co_return std::move(*available);
        }

        // 超时，抛出异常
        throw acquire_timeout_error();
    }

    // 归还连接
    auto release(std::string_view endpoint, tcp::socket socket)
        -> net::awaitable<void> {

        // 检查连接有效性
        if (!is_valid(socket)) {
            destroy_connection(socket);
            notify_waiters(endpoint);
            co_return;
        }

        // 放入空闲队列
        add_to_idle(endpoint, std::move(socket));

        // 通知等待者
        notify_waiters(endpoint);
    }

private:
    // 等待队列使用 strand 保证顺序
    using wait_queue = std::queue<net::steady_timer>;
    std::unordered_map<std::string, wait_queue> waiters_;
    std::unordered_map<std::string, std::deque<tcp::socket>> idle_;
};
```

### 健康检查策略

连接池需要检测连接有效性，健康检查策略影响性能和可靠性：

**获取时检查（validate_on_acquire）**：
- 每次获取连接前检查有效性
- 优点：确保获取的连接有效
- 缺点：增加获取延迟
- 适用：可靠性优先场景

**归还时检查（validate_on_release）**：
- 归还连接时检查有效性
- 优点：及时发现无效连接
- 缺点：增加归还延迟
- 适用：平衡性能和可靠性

**定期检查（background validation）**：
- 后台定期扫描检查连接
- 优点：不影响获取归还延迟
- 缺点：可能使用无效连接
- 适用：高性能场景

**检查方法**：
- TCP：尝试发送心跳数据或检查 socket 状态
- TLS：检查 session 状态
- HTTP：发送 OPTIONS 或 HEAD 请求
- 数据库：执行 SELECT 1 或 ping

### 连接预热与预分配

连接预热在池创建时预先建立连接，避免首次请求的连接建立延迟：

**预热策略**：
- 创建时立即建立 min_connections 个连接
- 后台异步预热，不阻塞主线程
- 预热失败不影响正常获取

**预分配资源**：
- 预分配 socket 句柄池
- 预分配缓冲区内存
- 预分配 SSL session

**预热时机**：
- 服务启动时
- 配置变更后
- 低负载时补充

### 统计与监控

连接池应提供详细的统计信息：

**基础统计**：
| 指标 | 说明 |
|------|------|
| active_count | 当前活跃连接数 |
| idle_count | 当前空闲连接数 |
| total_count | 总连接数 |
| pending_count | 等待获取的请求数 |

**性能统计**：
| 指标 | 说明 |
|------|------|
| hit_rate | 池命中率（空闲连接获取比例） |
| acquire_avg_ms | 平均获取时间 |
| acquire_max_ms | 最大获取时间 |
| wait_count | 等待获取的次数 |

**健康统计**：
| 指标 | 说明 |
|------|------|
| create_count | 创建连接次数 |
| destroy_count | 销毁连接次数 |
| validation_fail_count | 验证失败次数 |
| timeout_count | 获取超时次数 |

### 分片连接池

高并发场景下，单一连接池的锁竞争成为瓶颈，分片连接池可解决此问题：

**分片策略**：
- 按端点哈希分片
- 按线程 ID 分片
- 按 CPU 核心 ID 分片

**分片实现**：
```cpp
template<typename Connection>
class sharded_connection_pool {
public:
    auto acquire(std::string_view endpoint)
        -> net::awaitable<Connection> {
        // 根据端点哈希选择分片
        auto shard_id = hash(endpoint) % shard_count_;
        co_return co_await shards_[shard_id].acquire(endpoint);
    }

private:
    std::vector<connection_pool<Connection>> shards_;
    size_t shard_count_;
};
```

**优势**：
- 减少锁竞争
- 提高并发吞吐
- 更好的内存局部性

## 在 Prism 中的应用

Prism 作为代理服务器，连接池是核心组件，用于管理上游服务器连接。

### 连接池模块

Prism 在 `channel/connection/pool` 模块实现连接池：

**核心类**：
- `pool`：连接池主类，管理连接获取归还
- `entry`：连接条目，包含 socket 和元数据
- `factory`：连接工厂，创建新连接

### 配置选项

Prism 连接池配置：
```json
{
    "pool": {
        "max_cache_per_endpoint": 32,
        "max_idle_seconds": 60,
        "connect_timeout_ms": 5000,
        "recv_buffer_size": 65536,
        "send_buffer_size": 65536
    }
}
```

**参数说明**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| max_cache_per_endpoint | 32 | 每个端点最大缓存连接数 |
| max_idle_seconds | 60 | 空闲连接超时（秒） |
| connect_timeout_ms | 5000 | 连接建立超时（毫秒） |
| recv_buffer_size | 65536 | 接收缓冲区大小（字节） |
| send_buffer_size | 65536 | 发送缓冲区大小（字节） |

### 实现细节

**连接条目**：
```cpp
struct connection_entry {
    tcp::socket socket;
    endpoint_id endpoint;
    steady_clock::time_point last_used;
    steady_clock::time_point created;
    bool is_valid;
};
```

**连接池核心**：
```cpp
class connection_pool {
public:
    auto acquire(endpoint_id const& ep)
        -> net::awaitable<tcp::socket> {

        // 查找端点池
        auto& pool = get_endpoint_pool(ep);

        // 尝试获取空闲连接
        if (!pool.idle.empty()) {
            auto entry = pool.idle.back();
            pool.idle.pop_back();
            pool.active.insert(entry);
            co_return std::move(entry.socket);
        }

        // 池未满，创建新连接
        if (pool.total() < config_.max_cache_per_endpoint) {
            auto socket = co_await create_connection(ep);
            pool.active.insert(make_entry(socket, ep));
            co_return std::move(socket);
        }

        // 池满，等待
        co_await wait_for_connection(ep);
    }

    auto release(tcp::socket socket, endpoint_id const& ep)
        -> net::awaitable<void> {

        auto& pool = get_endpoint_pool(ep);

        // 移除活跃记录
        pool.active.erase(socket);

        // 检查有效性
        if (is_valid(socket)) {
            // 放入空闲队列
            pool.idle.push_back(make_entry(socket, ep));
        } else {
            // 销毁无效连接
            co_await close_connection(socket);
        }

        // 通知等待者
        notify_waiters(ep);
    }
};
```

### 清理机制

Prism 使用后台协程定期清理空闲连接：

```cpp
auto connection_pool::cleanup_loop()
    -> net::awaitable<void> {

    while (running_) {
        // 等待清理周期
        co_await cleanup_timer_.async_wait(net::use_awaitable);

        // 扫描所有端点池
        for (auto& [ep, pool] : endpoint_pools_) {
            auto now = steady_clock::now();
            auto cutoff = now - seconds(config_.max_idle_seconds);

            // 清理空闲超时的连接
            auto it = pool.idle.begin();
            while (it != pool.idle.end()) {
                if (it->last_used < cutoff) {
                    co_await close_connection(it->socket);
                    it = pool.idle.erase(it);
                } else {
                    ++it;
                }
            }
        }
    }
}
```

### 与路由器集成

Prism 的连接池与 `resolve/router` 模块集成：

```cpp
auto router::connect(endpoint_id const& ep)
    -> net::awaitable<tcp::socket> {

    // 尝试从连接池获取
    auto cached = pool_.acquire(ep);
    if (cached) {
        return std::move(*cached);
    }

    // DNS 解析
    auto addresses = co_await resolver_.resolve(ep.host, ep.port);

    // Happy Eyeballs 连接竞速
    auto socket = co_await eyeball_.connect(addresses);

    // 归还连接池（可选）
    // 通常在使用后归还

    co_return socket;
}
```

### 相关模块链接

| 模块 | 功能 | 文档 |
|------|------|------|
| pool | 连接池 | [[core/channel/connection/pool|pool]] |
| router | 路由器 | [[core/resolve/router|router]] |
| eyeball | Happy Eyeballs | [[core/channel/eyeball|eyeball]] |
| tunnel | 双向转发 | [[core/pipeline/tunnel|tunnel]] |

## 最佳实践

### 连接池大小配置

**估算方法**：
- 理论公式：池大小 = 平均请求处理时间 × 请求速率
- 实际建议：CPU核心数 × 2 ~ CPU核心数 × 4
- 根据实际监控数据调整

**示例计算**：
- 平均请求处理时间：100ms
- 目标请求速率：1000 req/s
- 理论池大小：100ms × 1000 = 100

**过度配置风险**：
- 内存占用过大
- 维护开销增加
- 目标服务器压力

### 超时参数设置

**空闲超时（max_idle_seconds）**：
- 短（30-60s）：节省资源，但命中率降低
- 长（300-600s）：提高命中率，但占用资源
- 建议：根据请求间隔和连接建立开销平衡

**获取超时（acquire_timeout_ms）**：
- 短（100-500ms）：快速失败，但可能误拒绝
- 长（3000-5000ms）：容错性强，但延迟增加
- 建议：根据用户期望响应时间设置

**连接超时（connect_timeout_ms）**：
- WAN：5000-10000ms
- LAN：1000-3000ms
- 根据 RTT 和网络稳定性调整

### 健康检查策略选择

**高性能优先**：
- validate_on_acquire = false
- validate_on_release = false
- background_validation = true

**可靠性优先**：
- validate_on_acquire = true
- validate_on_release = true
- 验证间隔短

**平衡策略**：
- validate_on_acquire = false
- validate_on_release = true
- background_validation = true

### 多端点场景处理

**端点池上限**：
- 单端点池：32-64 连接
- 总池上限：1000-5000 连接
- 不活跃端点清理

**端点隔离**：
- 不同目标使用不同池
- 防止连接混用
- 独立配置参数

### 资源监控

**关键指标**：
- 活跃连接数趋势
- 空闲连接数趋势
- 命中率变化
- 等待队列长度

**告警阈值**：
- 活跃连接数接近上限：告警
- 命中率持续下降：告警
- 等待队列增长：告警

## 常见问题

### Q1: 为什么连接池命中率低？

**可能原因**：
1. 空闲超时过短，连接快速清理
2. 池大小过小，连接快速消耗
3. 流量波动大，空闲连接不够用
4. 目标服务器不支持持久连接

**解决方案**：
- 廞长空闲超时
- 增大池大小
- 分析流量模式，优化配置

### Q2: 连接池导致连接泄漏怎么办？

**症状**：
- 活跃连接数持续增长
- 不归还连接
- 最终资源耗尽

**排查方法**：
- 使用 RAII 管理连接生命周期
- 归还连接使用 guard 类
- 定期扫描未归还连接

**解决方案**：
```cpp
// 使用 guard 自动归还连接
class connection_guard {
public:
    connection_guard(pool& p, socket s, endpoint ep)
        : pool_(p), socket_(s), endpoint_(ep) {}

    ~connection_guard() {
        // 异步归还（需要在协程中）
        net::co_spawn(pool_.io_context(),
            pool_.release(std::move(socket_), endpoint_),
            net::detached);
    }

    tcp::socket& get() { return socket_; }

private:
    pool& pool_;
    tcp::socket socket_;
    endpoint endpoint_;
};
```

### Q3: 如何处理目标服务器重启？

**现象**：
- 连接池中的连接失效
- 请求失败或超时

**处理方式**：
- 健康检查发现无效连接
- 归还时验证失败销毁连接
- 重新建立连接

**优化**：
- 缩短验证间隔
- 启用获取时验证
- 记录失败次数触发清理

### Q4: 连接池与多路复用如何配合？

**场景**：
- 单连接上复用多个流
- 连接池提供连接
- 多路复用层管理流

**最佳实践**：
- 连接池管理物理连接
- 多路复用管理逻辑流
- 独立的生命周期管理

### Q5: TLS 连接池有什么特殊考虑？

**TLS 特点**：
- 握手开销大（CPU 和 RTT）
- Session 可复用
- Session 有过期时间

**优化策略**：
- 复用 TLS session
- 配置 session cache
- 注意 session 过期时间

## 排障指南

### 性能问题诊断

**步骤 1：检查连接池统计**
```cpp
// 打印连接池统计
void print_pool_stats(pool& p) {
    auto stats = p.get_stats();
    spdlog::info("Pool stats: active={}, idle={}, hit_rate={:.2f}%",
                 stats.active_count, stats.idle_count, stats.hit_rate * 100);
}
```

**步骤 2：分析命中率**
- 命中率 < 50%：配置需要优化
- 命中率 > 80%：配置合理
- 命中率波动大：流量不稳定

**步骤 3：检查等待队列**
```cpp
// 监控等待队列
if (stats.pending_count > 0) {
    spdlog::warn("Connection pool wait queue: {} pending", stats.pending_count);
}
```

### 连接泄漏排查

**步骤 1：定期检查活跃连接**
```cpp
// 后台扫描长时间活跃的连接
for (auto& entry : active_connections) {
    auto duration = now - entry.last_used;
    if (duration > seconds(300)) {
        spdlog::warn("Connection potentially leaked: {} active for {}s",
                     entry.id, duration.count());
    }
}
```

**步骤 2：强制归还超时连接**
- 设置最大使用时间
- 超时强制归还
- 记录告警日志

**步骤 3：使用连接追踪**
```cpp
// 记录连接获取归还
spdlog::debug("Connection acquire: id={}, endpoint={}, caller={}",
              conn_id, endpoint, caller_info);
spdlog::debug("Connection release: id={}, endpoint={}",
              conn_id, endpoint);
```

### 目标服务器问题

**步骤 1：检查连接成功率**
```cpp
// 统计连接建立成功率
auto success_rate = create_success / create_total;
if (success_rate < 0.9) {
    spdlog::warn("Connection creation success rate: {:.2f}%", success_rate * 100);
}
```

**步骤 2：分析失败原因**
- 超时：网络问题或服务器慢
- RST：服务器拒绝或不可达
- DNS 错误：解析失败

**步骤 3：隔离问题端点**
```cpp
// 对特定端点降级
if (endpoint_stats.fail_rate > 0.5) {
    spdlog::warn("Endpoint {} has high fail rate, reducing pool size", endpoint);
    reduce_pool_size(endpoint);
}
```

### Prism 特定排障

**检查配置**：
```bash
# 查看当前配置
cat configuration.json | jq '.pool'
```

**启用详细日志**：
```json
{
    "trace": {
        "log_level": "debug",
        "modules": ["pool"]
    }
}
```

**监控端点池状态**：
```cpp
// 遍历所有端点池
for (auto& [ep, pool] : pool_registry) {
    spdlog::info("Endpoint {}: active={}, idle={}",
                 ep.to_string(), pool.active.size(), pool.idle.size());
}
```

## 参考资料

- [Connection Pool Pattern](https://en.wikipedia.org/wiki/Connection_pool)
- [HikariCP - JDBC Connection Pool](https://github.com/brettwooldridge/HikariCP)
- [Apache HttpClient Connection Management](https://hc.apache.org/httpclient-3.x/connection-management.html)

## 相关知识

- [[ref/network/tcp|TCP]] — 传输控制协议
- [[ref/network/happy-eyeballs|Happy Eyeballs]] — 连接竞速算法
- [[core/channel/connection/pool|Pool]] — Prism 连接池实现
- [[core/resolve/router|Router]] — Prism 路由器模块
- [[core/multiplex/duct|Duct]] — TCP 流多路复用