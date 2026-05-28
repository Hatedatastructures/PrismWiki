---
layer: core
source: "I:/code/Prism/include/prism/protocol/shadowsocks/salts.hpp"
title: SS2022 Salt 重放保护池
tags: [protocol, shadowsocks, salts, replay, pool, fnv1a, heterogeneous]
---

# SS2022 Salt 重放保护池

> 源码位置: `I:/code/Prism/include/prism/protocol/shadowsocks/salts.hpp`

## 概述

SS2022 Salt 重放保护池。SIP022 规范要求精确匹配的 salt 重放检测，禁止使用 Bloom filter。每个 salt 在 TTL 内只能出现一次。该池为 worker 线程独占，无需锁。

## 命名空间

```cpp
namespace psm::protocol::shadowsocks
```

## 类定义

### salt_pool

Salt 重放检测池，维护已见 salt 的精确集合，配合 TTL 自动过期清理。

```cpp
class salt_pool
```

**特性**:
- 每个 worker 线程持有独立实例（thread_local）
- 无需线程同步
- 使用异构查找避免 string 构造开销

## 构造函数

```cpp
explicit salt_pool(std::int64_t ttl_seconds = 60)
```

**参数**:
- `ttl_seconds`: Salt 条目的生存时间（默认 60 秒）

## 核心方法

### check_and_insert

检查并插入 salt。

```cpp
auto check_and_insert(std::span<const std::uint8_t> salt) -> bool;
```

**参数**:
- `salt`: Salt 数据

**返回**: `true` 表示首次出现（已插入），`false` 表示重放

**实现逻辑**:
1. 分摊清理：仅当距上次清理超过 1 秒时才执行
2. 异构查找：直接用 string_view 指向 salt 原始字节，零分配
3. 检查条目是否存在并是否过期

### cleanup

清理过期条目。

```cpp
void cleanup();
```

遍历条目，删除已过期的记录。

## 内部实现

### string_hash

异构查找哈希器，允许用 `string_view` 查找，无需构造 `string`。

```cpp
struct string_hash
{
    using is_transparent = void;
    auto operator()(std::string_view sv) const noexcept -> std::size_t
    {
        // FNV-1a hash
        std::size_t h = 0xcbf29ce484222325ULL;
        for (char c : sv)
        {
            h ^= static_cast<std::size_t>(c);
            h *= 0x100000001b3ULL;
        }
        return h;
    }
};
```

## 内部成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `entries_` | `std::unordered_map<std::string, time_point, string_hash, std::equal_to<>>` | Salt 条目映射 |
| `ttl_` | `std::chrono::seconds` | 条目生存时间 |
| `last_cleanup_` | `std::chrono::steady_clock::time_point` | 上次清理时间 |

## 调用链

- [[core/protocol/shadowsocks/relay|Relay]] - 中继器使用此池进行重放保护
- [[core/protocol/shadowsocks/config|Config]] - 配置中包含 salt_pool_ttl 参数
- [[core/protocol/shadowsocks/replay|Replay]] - UDP PacketID 重放检测