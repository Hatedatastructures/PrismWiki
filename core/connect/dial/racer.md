---
layer: core
source: include/prism/connect/dial/racer.hpp
title: address_racer
tags: [connect, dial, racer, happy-eyeballs, rfc8305, concurrent]
---

# address_racer

Happy Eyeballs (RFC 8305) 并发竞速连接器，实现多地址并发连接尝试。

## 概述

`address_racer` 实现 RFC 8305 的地址竞速算法，当 DNS 返回多个 IP 地址时并发尝试连接，第一个成功的连接 wins。该算法可有效降低连接延迟并保持 IPv6 优先。

### RFC 8305 核心机制

```
时间线:
  0ms    : 尝试连接 IP1 (IPv6 优先)
  250ms  : 尝试连接 IP2
  500ms  : 尝试连接 IP3
  ...

第一个成功的连接被返回，其他连接尝试被取消。
```

## 类定义

```cpp
class address_racer
{
public:
    explicit address_racer(connection_pool &pool);

    [[nodiscard]] auto race(std::span<const tcp::endpoint> endpoints)
        -> net::awaitable<pooled_connection>;

private:
    struct race_context;  // 竞速共享状态

    connection_pool &pool_;
    static constexpr auto secondary_delay = std::chrono::milliseconds(250);

    auto race_endpoint(tcp::endpoint ep, std::chrono::milliseconds delay,
                       std::shared_ptr<race_context> ctx)
        -> net::awaitable<void>;
};
```

## 核心方法

### race

```cpp
[[nodiscard]] auto race(std::span<const tcp::endpoint> endpoints)
    -> net::awaitable<pooled_connection>;
```

并发竞速连接多个端点。

**算法流程：**

1. 空列表检查：`endpoints` 为空返回空连接
2. 单端点优化：只有一个端点时直接 `pool_.async_acquire()`
3. 多端点竞速：第一个端点立即开始，后续按 250ms 间隔依次启动，第一个成功的 wins

**参数：**
- `endpoints`：候选端点列表，通常按优先级排序（IPv6 优先）

**返回值：** 成功连接，或空连接（全部失败时）

## 设计决策

### 为什么延迟间隔是 250ms？

RFC 8305 建议 250ms，基于三点：IPv6 先发优势鼓励 IPv6 使用；IPv6 不可达时快速回退到 IPv4；避免连接失败后的快速重试导致网络拥塞。

### 为什么不需要互斥锁？

协程在单线程 `io_context` 上执行，winner 写入与 timer cancel 之间没有挂起点（`co_await`），不存在竞态条件。

### 为什么子协程捕获连接池引用而非 this？

`address_racer` 通常是 `retry_connect()` 中的局部变量。`race()` 通过 `co_spawn` 启动子协程后，子协程可能比 racer 存活更久。如果子协程捕获 `this`，racer 析构后访问悬挂指针。捕获 `pool_` 引用（由 worker 持有）确保安全。

**后果**: 子协程不能访问 racer 的其他成员（如 `secondary_delay`），需通过参数传递。

## 约束

### 连接池生命周期

**类型**: 生命周期

**规则**: `pool_` 为非拥有引用，连接池的生命周期必须覆盖竞速操作。

**违反后果**: 子协程通过悬挂引用访问已销毁的连接池，崩溃。

**源码依据**: `racer.hpp:63`

### 单线程执行

**类型**: 线程安全

**规则**: 所有方法必须在同一个 io_context 线程上调用。winner 写入的无锁安全依赖单线程执行保证。

**违反后果**: 跨线程并发调用 `race()` 导致 race_context 数据竞争。

**源码依据**: `racer.hpp:33`

## 故障场景

### 所有候选地址连接失败

**触发条件**: 目标域名的所有解析结果（IPv4 + IPv6）都无法建立 TCP 连接。常见原因：目标宕机、防火墙阻断、本地网络中断。

**传播路径**: `race()` → 每个候选连接超时或拒绝 → 所有 `pending` 递减到 0 且无 winner → 返回无效 `pooled_connection` → `retry_connect()` → `async_forward()` 返回 `bad_gateway` → `dial()` 返回错误 → session 关闭 → 客户端收到 RST。

**外部表现**: 客户端连接超时。日志出现 `route failed: bad_gateway, target: xxx:yyy`。

**恢复机制**: 自动。下次请求重新触发 DNS 解析和竞速。临时网络故障恢复后新请求可成功。

**日志关键字**: `bad_gateway` + `route failed`

## 调用链

```
retry_connect() → address_racer → race()
    |
    +-- race_endpoint(ep1, 0ms, ctx)
    +-- race_endpoint(ep2, 250ms, ctx)  ← co_spawn
    +-- race_endpoint(ep3, 500ms, ctx)
         |
         v
    pool_.async_acquire(endpoint)
```

## 与其他模块的关系

- [[core/connect/dial/dial]]：通过 `retry_connect()` 调用 `race()`
- [[core/connect/pool/pool]]：连接池，提供 `async_acquire()` 方法
- [[core/resolve/dns/dns]]：DNS 解析器，提供候选端点列表