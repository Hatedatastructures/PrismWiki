---
title: "racer — Happy Eyeballs 并发竞速连接器"
source: "include/prism/channel/eyeball/racer.hpp"
module: "channel"
type: api
tags: [channel, eyeball, racer, happy-eyeballs, rfc8305, 竞速]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/connection/pool
  - channel/transport/reliable
  - ref/network/happy-eyeballs
  - ref/network/tcp
  - ref/programming/boost-asio
  - ref/programming/c++23-coroutines
---

# racer.hpp

> 源码: `include/prism/channel/eyeball/racer.hpp`
> 实现: `src/prism/channel/eyeball/racer.cpp`
> 模块: [[channel|channel]]

## 概述

[[ref/network/happy-eyeballs|Happy Eyeballs]] (RFC 8305) 并发竞速连接器。当 DNS 返回多个 IP 地址时并发尝试连接，第一个成功的连接 wins。

核心算法：
- 第一个端点立即尝试
- 后续端点延迟 250ms 后依次尝试
- 第一个成功的连接被返回，其他连接尝试被取消
- 可有效降低连接延迟并保持 IPv6 优先

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[channel/connection/pool|pool]] | 使用 `connection_pool` 和 `pooled_connection` |
| 被依赖 | 路由器 | 路由器使用竞速器建立连接 |

## 命名空间

`psm::channel::eyeball`

## 类: address_racer

> 源码: `include/prism/channel/eyeball/racer.hpp:50`

### 概述

Happy Eyeballs 地址竞速器。实现 [[ref/network/happy-eyeballs|RFC 8305]] 的并发连接算法。

设计约束：
- 不是线程安全的，应在单个 strand 中使用
- 子协程直接捕获连接池引用而非 this（racer 可能是局部变量）
- 单线程 `io_context` 上 winner 写入与 timer cancel 之间无挂起点，不需要互斥锁

### 构造函数

---

#### address_racer(pool)

**功能说明**：构造竞速器，接收连接池引用。竞速器不拥有连接池，调用方需确保连接池的生命周期覆盖竞速操作的整个周期。

**签名**：
```cpp
explicit address_racer(connection_pool &pool);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `pool` | `connection_pool &` | 连接池引用 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：路由器创建竞速器

**涉及的知识域**：[[ref/network/happy-eyeballs|Happy Eyeballs]]

### 成员函数

---

#### race()

**功能说明**：并发竞速连接多个端点。单端点直接连接，多端点按 250ms 间隔依次启动，第一个成功的连接被返回，其他连接归还到连接池。

**签名**：
```cpp
[[nodiscard]] auto race(std::span<const tcp::endpoint> endpoints)
    -> net::awaitable<pooled_connection>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `endpoints` | `std::span<const tcp::endpoint>` | 候选端点列表，通常按优先级排序 |

**返回值**：`net::awaitable<pooled_connection>` — 成功连接，或空连接（全部失败时）

**调用（向下）**：
- `pool_.async_acquire()` — 单端点时直接获取
- `race_endpoint()` — 多端点时并发竞速
- `co_spawn()` — 为每个端点投递协程

**被调用（向上）**：路由器建立上游连接时

**涉及的知识域**：[[ref/network/happy-eyeballs|Happy Eyeballs]], [[ref/programming/c++23-coroutines|C++23 协程]]

**race 流程**：
1. 空端点列表 → 返回空连接
2. 单端点 → 直接 `pool_.async_acquire()` 返回
3. 多端点 → 限制最大并发数为 6（避免 `WSAENOBUFS`）
4. 创建 `race_context` 共享状态
5. 为每个端点 `co_spawn` 竞速协程，第 1 个立即、后续按 250ms * i 延迟
6. 等待 `ctx->signal`（有获胜者或全部失败时被 cancel）
7. 返回 `ctx->result`

---

#### race_endpoint() (私有)

**功能说明**：单端点竞速协程。等待 staggered delay 后发起连接，连接成功时通过原子 `exchange` 竞争 winner 资格。

**签名**：
```cpp
auto race_endpoint(tcp::endpoint ep, std::chrono::milliseconds delay, std::shared_ptr<race_context> ctx)
    -> net::awaitable<void>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `ep` | `tcp::endpoint` | 目标端点 |
| `delay` | `std::chrono::milliseconds` | 启动延迟（0ms 为立即，250ms*i 为后续端点） |
| `ctx` | `std::shared_ptr<race_context>` | 竞速共享状态 |

**返回值**：`net::awaitable<void>`

**调用（向下）**：
- `net::steady_timer::async_wait()` — 等待延迟
- `pool_.async_acquire()` — 获取连接

**被调用（向上）**：`race()` 通过 `co_spawn` 启动

**涉及的知识域**：[[ref/programming/c++23-coroutines|C++23 协程]]

**race_endpoint 流程**：
1. delay > 0 → 等待定时器
2. 定时器被 cancel（已有获胜者）→ 跳过
3. `ctx->winner` 已为 true → 跳过
4. `pool_.async_acquire(ep)` 获取连接
5. 连接失败 → `ctx->complete()` 退出
6. `ctx->winner.exchange(true)` 原子竞争：
   - 得到 false（获胜）→ 保存连接，`ctx->signal.cancel()` 唤醒主协程
   - 得到 true（落败）→ `conn.reset()` 归还连接到池中

### 内部结构: race_context

竞速共享状态（定义在 `racer.cpp` 中）。

| 成员 | 类型 | 说明 |
|------|------|------|
| `winner` | `std::atomic<bool>` | 获胜标志 |
| `result` | `pooled_connection` | 获胜连接 |
| `pending` | `std::atomic<std::size_t>` | 未完成计数 |
| `signal` | `net::steady_timer` | 完成信号定时器 |

---

#### race_context::complete()

**功能说明**：递减 `pending` 计数，最后一个完成时取消 `signal` 定时器唤醒主协程。

**签名**：
```cpp
void complete();
```

**参数**：无

**返回值**：无

**调用（向下）**：`signal.cancel()`

**被调用（向上）**：`race_endpoint()` 结束时

**涉及的知识域**：[[ref/programming/c++23-coroutines|C++23 协程]]

### 私有成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `pool_` | `connection_pool &` | 连接池引用 |
| `secondary_delay` | `chrono::milliseconds` | 后续端点延迟（250ms，[[ref/network/happy-eyeballs|RFC 8305]] 建议值） |

### 调用链

```
router::async_forward → address_racer::race(endpoints)
  → race_endpoint(ep, 0ms, ctx)      // 第一个端点立即
  → race_endpoint(ep, 250ms, ctx)    // 后续端点延迟
  → race_endpoint(ep, 500ms, ctx)
  → ...
  → 第一个成功 → exchange 竞争 winner → signal.cancel()
  → 主协程唤醒 → 返回 pooled_connection
  → 落败者 conn.reset() → 归还到连接池
```
