---
layer: core
source: include/prism/account/entry.hpp
title: Account Entry 与 Lease
module: account
tags:
  - entry
  - lease
  - RAII
  - atomic
  - traffic-stats
created: 2026-05-23
updated: 2026-05-23
---

# Account Entry 与 Lease

> 源码位置: `include/prism/account/entry.hpp`

## 概述

`entry.hpp` 定义了账户运行时状态结构和连接租约的 RAII 封装，是 [[core/account/overview|Account 模块]] 的核心数据结构。

## entry 结构体

`entry` 存储单个账户的连接限制和实时统计信息，所有统计字段使用原子操作保证线程安全。该结构体由 [[core/account/directory|directory]] 统一管理生命周期，通过 `shared_ptr` 在多连接间共享。

```cpp
namespace psm::account
{
    struct entry
    {
        std::uint32_t max_connections{0};           // 最大连接数，0 表示不限制
        std::atomic_uint64_t uplink_bytes{0};       // 上行流量字节数
        std::atomic_uint64_t downlink_bytes{0};     // 下行流量字节数
        std::atomic_uint32_t active_connections{0}; // 当前活跃连接数
    };
}
```

### 字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_connections` | `uint32_t` | 0 | 最大并发连接数，0 表示无限制 |
| `uplink_bytes` | `atomic_uint64_t` | 0 | 客户端→服务端方向累计流量 |
| `downlink_bytes` | `atomic_uint64_t` | 0 | 服务端→客户端方向累计流量 |
| `active_connections` | `atomic_uint32_t` | 0 | 当前活跃的连接数量 |

### 原子操作与内存序

所有统计字段使用 `std::memory_order_relaxed` 进行原子操作：

- **原因**: 流量统计和连接计数仅用于近似监控，不参与同步控制
- **好处**: relaxed 序在 x86 上几乎零开销，避免完整的内存屏障
- **注意**: 如果需要精确的统计快照（读取时所有字段一致），需要外部同步机制

### 生命周期管理

`entry` 通过 `std::shared_ptr<entry>` 管理生命周期：

```
loader 创建 entry
    │
    ▼
directory::upsert() 或 insert()
    │
    ├── 存储 shared_ptr<entry> 到映射表
    │
    ▼
try_acquire() 查找并返回 shared_ptr<entry>
    │
    ▼
lease 持有 shared_ptr<entry>
    │
    ├── 多个 lease 可共享同一 entry
    │   (多协议凭证共享连接配额)
    │
    ▼
所有 lease 析构 → entry 引用计数归零 → 自动销毁
```

## lease 类

`lease` 是连接租约的 RAII 封装，持有 `entry` 的共享所有权。构造时假定活跃连接数已递增，析构时自动递减活跃连接数，确保连接异常退出时不会泄漏连接计数。

### 类定义

```cpp
namespace psm::account
{
    class lease
    {
    public:
        lease() = default;
        explicit lease(std::shared_ptr<entry> state) noexcept;
        lease(lease &&other) noexcept;
        auto operator=(lease &&other) noexcept -> lease &;

        // 禁止拷贝
        lease(const lease &) = delete;
        auto operator=(const lease &) -> lease & = delete;

        ~lease(); // 调用 release()

        // 访问器
        [[nodiscard]] auto get() const noexcept -> entry *;
        [[nodiscard]] explicit operator bool() const noexcept;

    private:
        void release() noexcept;
        std::shared_ptr<entry> state_;
    };
}
```

### 构造函数

```cpp
explicit lease(std::shared_ptr<entry> state) noexcept;
```

接管账户状态指针的所有权。**不自动递增连接数** -- 调用方需确保在构造租约前已递增 `active_connections`（通常通过 `try_acquire()` 完成）。

### 移动语义

```cpp
lease(lease &&other) noexcept;
auto operator=(lease &&other) noexcept -> lease &;
```

支持移动构造和移动赋值，移动后源对象为空（`state_ == nullptr`）。移动赋值会先释放当前持有的租约（递减连接计数），再转移源租约所有权。

### 析构与释放

```cpp
~lease();
```

析构时调用 `release()` 方法：

```cpp
void release() noexcept
{
    if (!state_) return;
    state_->active_connections.fetch_sub(1, std::memory_order_relaxed);
    state_.reset();
}
```

- 空租约调用为空操作
- 使用 relaxed 内存序递减连接计数
- 递减后重置 `shared_ptr`，可能触发 `entry` 销毁

### 访问器

```cpp
[[nodiscard]] auto get() const noexcept -> entry *;
```

返回底层账户状态的裸指针。租约为空时返回 `nullptr`。用于向流量累加函数传递。

```cpp
[[nodiscard]] explicit operator bool() const noexcept;
```

检查租约是否有效（内部 `shared_ptr` 非空）。`explicit` 防止隐式转换。

## 自由函数

### accumulate_uplink

```cpp
inline void accumulate_uplink(entry *state, const std::uint64_t bytes) noexcept;
```

累加上行流量。空指针时为空操作，使用 relaxed 内存序原子递增 `uplink_bytes`。

### accumulate_downlink

```cpp
inline void accumulate_downlink(entry *state, const std::uint64_t bytes) noexcept;
```

累加下行流量。空指针时为空操作，使用 relaxed 内存序原子递增 `downlink_bytes`。

## 使用模式

### 典型会话流程

```cpp
// 1. 尝试获取租约
auto user_lease = account::try_acquire(account_dir, credential);
if (!user_lease) {
    // 认证失败或连接数已满
    close_connection();
    return;
}

// 2. 数据传输中累加流量
auto *state = user_lease.get();
account::accumulate_uplink(state, bytes_sent);
account::accumulate_downlink(state, bytes_received);

// 3. 会话结束，租约自动析构
// ~lease() → active_connections.fetch_sub(1)
```

### 移动语义使用

```cpp
// lease 可安全移动，用于所有权转移
auto handle_session(account::lease lease) -> net::awaitable<void>
{
    // lease 通过参数移动传入
    // ... 处理会话 ...
    co_return;
    // 函数返回时 lease 析构，递减连接计数
}

// 调用方
co_await handle_session(std::move(user_lease));
```

## 调用链

```
try_acquire()
    ├── directory::find() → shared_ptr<entry>
    ├── CAS 递增 active_connections
    └── 返回 lease(shared_ptr<entry>)
            │
            ▼
        session 持有 lease
            │
            ├── get() → entry*
            │       ├── accumulate_uplink(entry*, bytes)
            │       └── accumulate_downlink(entry*, bytes)
            │
            ▼
        ~lease()
            └── release()
                  └── active_connections.fetch_sub(1)
```

## 设计考量

### 为什么 lease 不自动递增

`lease` 构造函数不递增 `active_connections`，而是要求调用方在构造前完成递增。这是因为在多线程环境下，递增和构造必须是原子的：

```cpp
// try_acquire() 中的 CAS 循环
auto current = entry_ptr->active_connections.load(relaxed);
while (true) {
    if (current >= max_connections) return {};  // 已满
    if (CAS(current, current + 1)) {
        return lease(std::move(entry_ptr));  // 递增成功后才构造 lease
    }
}
```

如果 `lease` 构造时递增，CAS 成功后、构造前的间隙可能导致计数不一致。

### 为什么使用 relaxed 内存序

`uplink_bytes`、`downlink_bytes` 和 `active_connections` 仅用于近似统计和连接限制，不参与任何同步：

- 流量统计: 用于监控和日志，精确性要求低
- 连接计数: 连接限制是"尽力而为"的软限制，少量误差可接受
- relaxed 在 x86 上等同于普通读写，零额外开销

## 相关文档

- [[core/account/overview|Account 模块总览]] - 模块架构与调用链
- [[core/account/directory|directory 详解]] - 目录管理与查找
- [[core/context/context|Context]] - session 上下文中的 account_lease 字段
- [[core/memory/container|Memory Container]] - PMR 容器定义
- [[core/trace/overview|Trace]] - 流量统计日志输出
