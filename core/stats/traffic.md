---
title: Traffic 流量统计
layer: core
source:
  - I:/code/Prism/include/prism/stats/traffic.hpp
  - I:/code/Prism/src/prism/stats/traffic.cpp
module: stats
tags: [stats, traffic, protocol, cow, performance]
created: 2026-05-23
updated: 2026-05-23
---

# Traffic 流量统计

Traffic 子模块提供 per-worker 流量统计和全局聚合能力。每个 [[core/instance/worker/worker|Worker]] 持有一个 `traffic_state` 实例，在热路径中以批量刷入方式记录流量数据。全局聚合使用 Copy-on-Write (COW) 注册表，按需汇总所有 worker 的流量计数器。

> 源码: `include/prism/stats/traffic.hpp` | 实现: `src/prism/stats/traffic.cpp`

## 类定义

### protocol_slot

```cpp
struct alignas(64) protocol_slot
{
    std::atomic<std::uint64_t> connections{0};
    std::atomic<std::uint64_t> active{0};
    std::atomic<std::uint64_t> uplink_bytes{0};
    std::atomic<std::uint64_t> downlink_bytes{0};
};
```

### traffic_state

```cpp
class alignas(64) traffic_state
{
public:
    traffic_state() = default;

    void on_connect() noexcept;
    void on_protocol_detected(protocol::protocol_type type) noexcept;
    void on_disconnect(protocol::protocol_type type) noexcept;
    void flush_traffic(protocol::protocol_type proto, std::uint64_t up, std::uint64_t down) noexcept;
    void on_auth_success() noexcept;
    void on_auth_failure() noexcept;

    [[nodiscard]] auto snapshot() const noexcept -> traffic_snapshot;
    void reset() noexcept;

    static void register_instance(traffic_state *s) noexcept;
    static void unregister_instance(traffic_state *s) noexcept;
    [[nodiscard]] static auto aggregate() noexcept -> traffic_snapshot;

private:
    std::atomic<std::uint64_t> total_connections_{0};
    std::atomic<std::uint64_t> total_active_{0};
    std::atomic<std::uint64_t> total_uplink_{0};
    std::atomic<std::uint64_t> total_downlink_{0};
    protocol_slot protocols_[protocol_slot_count];
    std::atomic<std::uint64_t> auth_success_{0};
    std::atomic<std::uint64_t> auth_failure_{0};
};
```

## protocol_slot 详解

### 内存布局

```
┌──────────── 64 字节缓存行 ────────────┐
│ connections     (8B)  offset 0        │
│ active          (8B)  offset 8        │
│ uplink_bytes    (8B)  offset 8        │
│ downlink_bytes  (8B)  offset 24       │
│ padding         (32B)                 │
└───────────────────────────────────────┘
```

整个 `protocol_slot` 占 64 字节，通过 `alignas(64)` 独占缓存行。内部四个 `atomic<uint64_t>` 共享缓存行是安全的，因为：

1. **单写者保证**: 每个 `traffic_state` 只被所属的 worker 线程写入
2. **无跨核争用**: 不同 worker 的 `protocol_slot` 在不同的缓存行上
3. **读路径宽松**: `aggregate()` 只读取，不修改，不会导致缓存行失效

### 字段语义

| 字段 | 方向 | 含义 |
|------|------|------|
| `connections` | 仅增 | 历史总连接数，建立时递增，断开时不递减 |
| `active` | 增减 | 当前活跃连接数，建立时递增，断开时递减 |
| `uplink_bytes` | 仅增 | 上行（客户端→上游）累计字节数 |
| `downlink_bytes` | 仅增 | 下行（上游→客户端）累计字节数 |

## traffic_state 方法详解

### on_connect()

```cpp
void traffic_state::on_connect() noexcept
{
    total_connections_.fetch_add(1, std::memory_order::relaxed);
    total_active_.fetch_add(1, std::memory_order::relaxed);
}
```

**调用（向上）**: [[core/instance/worker/launch|launch::start()]] 中，新连接建立时调用。

**原子操作**: 2 次 `fetch_add`。递增全局总连接数和活跃连接数。

### on_protocol_detected()

```cpp
void traffic_state::on_protocol_detected(protocol::protocol_type type) noexcept
{
    const auto i = static_cast<std::uint8_t>(type);
    protocols_[i].connections.fetch_add(1, std::memory_order::relaxed);
    protocols_[i].active.fetch_add(1, std::memory_order::relaxed);
}
```

**调用（向上）**: `session::diversion()` 中，协议识别完成后调用。

**数组索引**: 将 `protocol_type` 枚举强转为 `uint8_t` 索引 `protocols_[]` 数组。参见 [[core/stats/snapshot|protocol_slot_count = 16]]。

### on_disconnect()

```cpp
void traffic_state::on_disconnect(protocol::protocol_type type) noexcept
{
    total_active_.fetch_sub(1, std::memory_order::relaxed);
    const auto i = static_cast<std::uint8_t>(type);
    protocols_[i].active.fetch_sub(1, std::memory_order::relaxed);
}
```

**调用（向上）**: `session::release_resources()` 中，连接断开时调用。

**注意**: 只递减 `active`，不递减 `connections`。`connections` 是历史累计值。

### flush_traffic()

```cpp
void traffic_state::flush_traffic(protocol::protocol_type proto,
                                   std::uint64_t up,
                                   std::uint64_t down) noexcept
{
    const auto i = static_cast<std::uint8_t>(proto);
    if (up)
    {
        total_uplink_.fetch_add(up, std::memory_order::relaxed);
        protocols_[i].uplink_bytes.fetch_add(up, std::memory_order::relaxed);
    }
    if (down)
    {
        total_downlink_.fetch_add(down, std::memory_order::relaxed);
        protocols_[i].downlink_bytes.fetch_add(down, std::memory_order::relaxed);
    }
}
```

**设计亮点 -- 批量刷入**:

热路径中的流量数据不逐字节刷入 `traffic_state`，而是在局部变量中累积，会话/子流结束时一次性调用 `flush_traffic()`。每次调用产生最多 4 次 `fetch_add`（有上行 + 有下行的情况）。

```
单次会话的统计操作开销：

建立连接:
  on_connect()              → 2 次 fetch_add
  on_protocol_detected()    → 2 次 fetch_add

数据传输阶段:
  (无统计操作，流量在局部变量累积)

会话结束:
  flush_traffic()           → 最多 4 次 fetch_add
  on_disconnect()           → 2 次 fetch_add

总计: 最多 10 次原子操作 / 会话
```

**调用（向上）**:
- [[core/connect/tunnel/tunnel|tunnel]] 双向转发结束时
- [[core/protocol/common/udp_relay|UDP relay]] 结束时
- [[core/multiplex/core|mux core]] 子流结束时

### on_auth_success() / on_auth_failure()

```cpp
void traffic_state::on_auth_success() noexcept
{
    auth_success_.fetch_add(1, std::memory_order::relaxed);
}

void traffic_state::on_auth_failure() noexcept
{
    auth_failure_.fetch_add(1, std::memory_order::relaxed);
}
```

**调用（向上）**: [[core/instance/worker/launch|launch::start()]] 的 `credential_function` 中调用。

### snapshot()

```cpp
auto traffic_state::snapshot() const noexcept -> traffic_snapshot
{
    traffic_snapshot s;
    s.total_connections = total_connections_.load(std::memory_order::relaxed);
    s.total_active = total_active_.load(std::memory_order::relaxed);
    s.total_uplink = total_uplink_.load(std::memory_order::relaxed);
    s.total_downlink = total_downlink_.load(std::memory_order::relaxed);
    s.auth_success = auth_success_.load(std::memory_order::relaxed);
    s.auth_failure = auth_failure_.load(std::memory_order::relaxed);

    for (std::size_t i = 0; i < protocol_slot_count; ++i)
    {
        s.protocols[i].connections = protocols_[i].connections.load(std::memory_order::relaxed);
        s.protocols[i].active = protocols_[i].active.load(std::memory_order::relaxed);
        s.protocols[i].uplink_bytes = protocols_[i].uplink_bytes.load(std::memory_order::relaxed);
        s.protocols[i].downlink_bytes = protocols_[i].downlink_bytes.load(std::memory_order::relaxed);
    }
    return s;
}
```

**开销**: 6 次标量 `load` + 16 x 4 = 64 次数组 `load` = 70 次原子读取。在 x86-64 上原子读取编译为普通 `mov` 指令，总计约 20ns。

## 全局聚合 -- COW 注册表

### 注册表设计

```cpp
using registry_vector = std::vector<traffic_state *>;
static std::atomic<registry_vector *> g_registry{nullptr};
```

全局注册表是一个原子指针，指向 `vector<traffic_state*>`。采用 Copy-on-Write 模式：修改时创建新 vector 替换旧的，旧的 vector 不释放（可能有并发的 `aggregate()` 在读）。

### register_instance()

```cpp
void traffic_state::register_instance(traffic_state *s) noexcept
{
    auto *old = load_registry();        // acquire
    auto *next = new registry_vector();
    if (old)
        *next = *old;                   // 拷贝旧注册表
    next->push_back(s);                 // 追加新实例
    store_registry(next);               // release
}
```

**调用（向上）**: [[core/instance/worker/worker|Worker 构造函数]] 中调用。

**COW 流程**:

```
注册表状态: [W0, W1, W2]

Worker 3 构造:
  1. 读取旧注册表: [W0, W1, W2]
  2. 创建新注册表: [W0, W1, W2, W3]
  3. 原子替换指针 → 新注册表
  4. 旧注册表 [W0, W1, W2] 不释放（可能有并发 aggregate 在读）

内存泄漏: 每次注册分配一个 vector（约 24 + 8*N 字节）
Worker 数量通常 < 64，泄漏总量 < 10KB，完全可忽略。
```

### aggregate()

```cpp
auto traffic_state::aggregate() noexcept -> traffic_snapshot
{
    auto *reg = load_registry();
    if (!reg) return {};
    traffic_snapshot result;
    for (auto *instance : *reg)
    {
        auto s = instance->snapshot();
        result.total_connections += s.total_connections;
        result.total_active += s.total_active;
        result.total_uplink += s.total_uplink;
        result.total_downlink += s.total_downlink;
        result.auth_success += s.auth_success;
        result.auth_failure += s.auth_failure;
        for (std::size_t i = 0; i < protocol_slot_count; ++i)
        {
            result.protocols[i].connections += s.protocols[i].connections;
            result.protocols[i].active += s.protocols[i].active;
            result.protocols[i].uplink_bytes += s.protocols[i].uplink_bytes;
            result.protocols[i].downlink_bytes += s.protocols[i].downlink_bytes;
        }
    }
    return result;
}
```

**开销**: N 个 worker x (6 标量 load + 64 数组 load + 70 加法) = N x 140 操作。

对于 8 个 worker：1120 次操作，约 400ns。这远低于热路径的纳秒级延迟要求，因为 `aggregate()` 仅在管理接口或监控系统中调用（频率约 1 次/秒）。

## 流量注入点

各协议处理器通过 `set_traffic()` 方法获取 `traffic_state` 指针，在会话结束时调用 `flush_traffic()`：

```
协议处理器的流量注入路径：

[[core/protocol/trojan/conn|Trojan]]
  set_traffic(traffic_state*, protocol_type::trojan)
  └── 会话结束时 flush_traffic(trojan, up, down)

[[core/protocol/vless/conn|VLESS]]
  set_traffic(traffic_state*, protocol_type::vless)
  └── 会话结束时 flush_traffic(vless, up, down)

[[core/protocol/socks5/conn|SOCKS5]]
  set_traffic(traffic_state*, protocol_type::socks5)
  └── 会话结束时 flush_traffic(socks5, up, down)

[[core/multiplex/core|Mux Core]]
  set_traffic(traffic_state*, outer_protocol_type)
  └── 子流结束时 flush_traffic(outer_proto, mux_up, mux_down)
```

## traffic_state 内存布局

```
sizeof(traffic_state) ≈ 1184 字节（alignas(64)）

┌────── 64B ──────┐
│ total_connections_ │  8B
│ total_active_      │  8B
│ total_uplink_      │  8B
│ total_downlink_    │  8B
│ padding            │  32B
├────── 64B ──────┤  ← protocols_[0]
│ connections        │  8B
│ active             │  8B
│ uplink_bytes       │  8B
│ downlink_bytes     │  8B
│ padding            │  32B
├────── 64B ──────┤  ← protocols_[1]
│ ...                │
├────── 64B ──────┤
│ ...                │
├────── 64B ──────┤  ← protocols_[15]
│ ...                │
├────── 64B ──────┤  ← auth + padding
│ auth_success_      │  8B
│ auth_failure_      │  8B
│ padding            │  48B
└──────────────────┘
```

16 个 `protocol_slot` x 64 字节 + 全局计数器 + 认证计数器 = 约 1184 字节。

## 性能影响

### 热路径开销（单次会话）

| 操作 | 原子操作数 | 估计耗时 |
|------|-----------|---------|
| `on_connect()` | 2 fetch_add | ~12ns |
| `on_protocol_detected()` | 2 fetch_add | ~12ns |
| `flush_traffic()` | 4 fetch_add | ~24ns |
| `on_disconnect()` | 2 fetch_sub | ~12ns |
| **总计** | **10 原子操作** | **~60ns** |

单次会话的统计操作总开销约 60ns，相比于会话生命周期（通常 > 100ms），占比 < 0.0001%。

### 内存开销（8 worker）

| 组件 | 大小 |
|------|------|
| 8 x traffic_state | ~9.5KB |
| COW 注册表（9 个 vector，含初始 nullptr） | ~200 字节 |
| **总计** | **~10KB** |

## 相关文档

- [[core/stats/overview|Stats 模块总览]] -- 模块架构
- [[core/stats/snapshot|snapshot]] -- traffic_snapshot 和 protocol_snapshot 定义
- [[core/stats/counter|counter]] -- 底层原子计数器原语
- [[core/stats/runtime|runtime]] -- 互补的运行时统计
- [[core/instance/worker/worker|Worker]] -- traffic_state 的持有者
- [[core/protocol/trojan/conn|Trojan]] -- 协议处理器流量注入点
- [[core/multiplex/core|Mux Core]] -- 多路复用流量注入点
