---
title: config
layer: core
source: I:/code/Prism/include/prism/connect/pool/config.hpp
module: connect
tags:
  - config
  - connection-pool
  - tuning
  - performance
created: 2026-05-23
updated: 2026-05-23
---

# config

连接池配置结构体，控制 [[core/connect/pool/pool|connection_pool]] 的行为参数，包括缓存容量、超时策略、缓冲区大小和 TCP 选项。

## 概述

`config` 是一个简单的聚合结构体（所有字段均有默认值），作为 [[core/connect/pool/pool|connection_pool]] 的构造参数传入。配置在连接池创建时确定，运行期间不修改。

该结构体的设计目标是提供合理的默认值，同时允许运维根据实际场景调优。默认值经过基准测试验证，适合大多数生产环境。

## 结构体定义

```cpp
struct config
{
    std::uint32_t max_cache_per_endpoint = 32U;  // 单端点最大缓存连接数
    std::uint64_t connect_timeout_ms = 300ULL;   // 连接超时（毫秒）
    std::uint64_t max_idle_seconds = 30ULL;      // 空闲连接最大存活时间（秒）
    std::uint64_t cleanup_interval_sec = 10ULL;  // 后台清理间隔（秒）
    std::uint32_t recv_buffer_size = 65536U;     // 接收缓冲区大小（字节）
    std::uint32_t send_buffer_size = 65536U;     // 发送缓冲区大小（字节）
    bool tcp_nodelay = true;                     // 启用 TCP_NODELAY
    bool keep_alive = true;                      // 启用 SO_KEEPALIVE
    bool cache_ipv6 = false;                     // 是否缓存 IPv6 连接
};
```

## 字段详解

### max_cache_per_endpoint

```cpp
std::uint32_t max_cache_per_endpoint = 32U;
```

**说明：** 单个目标端点（IP:Port 组合）在连接池中最多缓存的空闲连接数。

**默认值：** 32

**影响：**
- 值越大，连接复用率越高，但内存占用也越大
- 值过小会导致频繁的新建/关闭连接，增加 TCP 握手开销
- 实际缓存数受 `max_idle_seconds` 和 `cleanup_interval_sec` 影响

**调优建议：**

| 场景 | 推荐值 | 原因 |
|------|--------|------|
| 高并发少量端点 | 64-128 | 提高复用率 |
| 低并发大量端点 | 16-32 | 减少内存压力 |
| 内存受限环境 | 8-16 | 牺牲复用率换内存 |

> [!warning]
> 设置过大的值可能导致内存压力，尤其在缓存大量不同端点时。每个空闲连接占用一个 socket fd 和内核缓冲区。

### connect_timeout_ms

```cpp
std::uint64_t connect_timeout_ms = 300ULL;
```

**说明：** 新建 TCP 连接的超时时间（毫秒）。

**默认值：** 300ms

**影响：**
- 超时后连接尝试被取消，返回 `fault::code::timeout`
- 该值直接影响不可达目标的感知延迟
- 在 [[core/connect/dial/racer|address_racer]] 竞速连接中，该超时作用于每个单独的连接尝试

**调优建议：**

| 场景 | 推荐值 | 原因 |
|------|--------|------|
| 内网/低延迟 | 100-200ms | 快速失败 |
| 公网/一般延迟 | 300-500ms | 平衡成功率与延迟 |
| 高延迟网络 | 500-1000ms | 避免误超时 |

### max_idle_seconds

```cpp
std::uint64_t max_idle_seconds = 30ULL;
```

**说明：** 空闲连接在连接池中的最大存活时间（秒）。超过此时间的连接在清理周期中被移除。

**默认值：** 30 秒

**影响：**
- 值越大，连接复用窗口越长，但持有僵尸连接的风险越高
- 值过小会导致频繁重建连接
- 与 `cleanup_interval_sec` 配合决定实际的过期精度

**工作机制：**
- 连接归还时记录 `last_used` 时间戳
- 后台清理协程按 `cleanup_interval_sec` 周期检查
- `steady_clock::now() - last_used > max_idle_seconds` 的连接被驱逐

### cleanup_interval_sec

```cpp
std::uint64_t cleanup_interval_sec = 10ULL;
```

**说明：** 后台清理协程的运行间隔（秒）。

**默认值：** 10 秒

**影响：**
- 间隔越短，过期连接清理越及时，但 CPU 开销越大
- 间隔越长，实际过期时间可能比 `max_idle_seconds` 延长最多一个间隔周期
- 实际最大空闲时间 = `max_idle_seconds + cleanup_interval_sec`

**工作机制：**
- 由 [[core/connect/pool/pool|connection_pool::start()]] 启动的后台协程驱动
- 使用 `net::steady_timer` 实现周期性调度
- 清理使用原地压缩算法，避免内存分配

### recv_buffer_size / send_buffer_size

```cpp
std::uint32_t recv_buffer_size = 65536U;  // 64KB
std::uint32_t send_buffer_size = 65536U;  // 64KB
```

**说明：** TCP socket 的接收/发送缓冲区大小（字节）。

**默认值：** 64KB（65536 字节）

**影响：**
- 值越大，单次 `async_read_some` / `async_write_some` 可处理的字节数越多
- 直接影响 [[core/connect/tunnel/tunnel|tunnel]] 的吞吐量
- 过小会导致频繁的系统调用，过大浪费内核内存

**调优建议：**
- 一般代理场景：64KB（默认值）
- 高吞吐量场景：128KB-256KB
- 内存受限环境：32KB

### tcp_nodelay

```cpp
bool tcp_nodelay = true;
```

**说明：** 是否对新建立的连接启用 `TCP_NODELAY` 选项（禁用 Nagle 算法）。

**默认值：** `true`（启用）

**影响：**
- `true`：禁用 Nagle 算法，小数据包立即发送，降低延迟
- `false`：启用 Nagle 算法，小数据包被缓冲合并，提高带宽利用率

**为什么默认启用：**
- 代理服务器是延迟敏感型应用
- 代理转发的数据通常已经过客户端的 Nagle 算法优化
- 二次 Nagle 会引入不必要的延迟

### keep_alive

```cpp
bool keep_alive = true;
```

**说明：** 是否对新建立的连接启用 `SO_KEEPALIVE` 选项。

**默认值：** `true`（启用）

**影响：**
- `true`：内核自动发送 TCP keepalive 探测包，检测死连接
- `false`：不发送探测包，死连接可能在连接池中长时间存在

**为什么默认启用：**
- 连接池复用的连接可能在空闲期间被中间设备（NAT、防火墙）丢弃
- keepalive 探测确保连接仍然有效
- 与 [[core/connect/pool/health|健康检测]] 配合，形成双层保障

### cache_ipv6

```cpp
bool cache_ipv6 = false;
```

**说明：** 是否在连接池中缓存 IPv6 连接。

**默认值：** `false`（不缓存）

**影响：**
- `false`：IPv6 连接归还时直接关闭，不进入缓存
- `true`：IPv6 连接正常缓存和复用

**为什么默认不缓存：**
- 许多环境中 IPv6 连接的稳定性低于 IPv4
- NAT64/DNS64 环境下 IPv6 连接可能有不同的超时行为
- 缓存不稳定的连接可能导致复用时失败

**IPv6 连接归还时的处理：**

```
recycle(socket, endpoint)
    |
    v
[cache_ipv6 == false && endpoint 是 IPv6?]
    |-- Yes --> 直接关闭 socket（不缓存）
    |-- No  --> 继续健康检测 + 容量检查 + 入栈缓存
```

## 配置映射

`config` 的字段与 JSON 配置文件的映射关系：

```json
{
  "pool": {
    "max_cache_per_endpoint": 32,
    "max_idle_seconds": 30,
    "connect_timeout_ms": 300,
    "recv_buffer_size": 65536,
    "send_buffer_size": 65536
  }
}
```

| config 字段 | JSON 路径 | 类型 |
|-------------|-----------|------|
| `max_cache_per_endpoint` | `pool.max_cache_per_endpoint` | uint32 |
| `connect_timeout_ms` | `pool.connect_timeout_ms` | uint64 |
| `max_idle_seconds` | `pool.max_idle_seconds` | uint64 |
| `cleanup_interval_sec` | (内部参数) | uint64 |
| `recv_buffer_size` | `pool.recv_buffer_size` | uint32 |
| `send_buffer_size` | `pool.send_buffer_size` | uint32 |
| `tcp_nodelay` | (内部参数) | bool |
| `keep_alive` | (内部参数) | bool |
| `cache_ipv6` | (内部参数) | bool |

> [!note]
> `cleanup_interval_sec`、`tcp_nodelay`、`keep_alive`、`cache_ipv6` 为内部调优参数，不暴露在 JSON 配置文件中。

## 配置使用位置

`config` 在以下组件中被使用：

- [[core/connect/pool/pool|connection_pool]]：构造时传入，控制池行为
- [[core/connect/dial/router|router]]：持有连接池的引用，间接使用配置
- [[core/connect/tunnel/tunnel|tunnel]]：通过 `context::session::buffer_size` 使用缓冲区配置

## 时序图：配置到运行时

```
配置文件 (JSON)
    |
    v
loader::load()
    |
    v
psm::config (全局配置)
    |
    v
worker 初始化
    |
    +-- connection_pool(ioc, mr, config)
    |       |
    |       v
    |   config_ 成员
    |       |
    |       +-- async_acquire() 使用 connect_timeout_ms, recv/send_buffer_size,
    |       |   tcp_nodelay, keep_alive
    |       |
    |       +-- recycle() 使用 max_cache_per_endpoint, cache_ipv6
    |       |
    |       +-- cleanup() 使用 max_idle_seconds
    |
    +-- router(pool, ioc, dns_cfg)
```

## 与其他模块的关系

- [[core/connect/pool/pool]]：config 的直接消费者
- [[core/connect/pool/health]]：健康检测在 recycle 中被调用，受 cache_ipv6 影响
- [[core/connect/dial/router]]：通过连接池间接使用 config
- [[core/connect/dial/dial]]：拨号函数通过路由器间接使用 config
- [[core/connect/tunnel/tunnel]]：隧道使用 buffer_size 配置
- [[core/loader]]：配置加载器，从 JSON 解析 config

## 注意事项

1. **聚合结构体**：`config` 是聚合体（无自定义构造函数），支持聚合初始化和大括号初始化
2. **默认值适用**：所有字段均有合理默认值，可使用 `config{}` 直接构造默认配置
3. **运行期不变**：config 在连接池创建时设置，运行期间不修改。热加载需要重建连接池
4. **类型安全**：所有数值字段使用固定宽度整数类型（`uint32_t`、`uint64_t`），避免平台差异
5. **单位明确**：时间字段使用 `_ms`（毫秒）和 `_sec`（秒）后缀，避免混淆
