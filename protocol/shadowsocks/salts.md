---
title: "salts — SS2022 Salt 重放保护池"
source: "include/prism/protocol/shadowsocks/salts.hpp"
module: "protocol"
type: api
tags: [protocol, shadowsocks, salts, 重放保护, TTL, FNV-1a]
related:
  - "[[protocol/shadowsocks/relay|relay]]"
  - "[[protocol/shadowsocks/replay|replay]]"
  - "[[protocol/shadowsocks/constants|constants]]"
created: 2026-05-15
updated: 2026-05-15
---

# salts.hpp

> 源码: `include/prism/protocol/shadowsocks/salts.hpp`
> 模块: [[protocol|protocol]] > shadowsocks

## 概述

SS2022 Salt 重放保护池。SIP022 规范要求精确匹配的 salt 重放检测，禁止使用 Bloom filter。每个 salt 在 TTL 内只能出现一次。该池为 worker 线程独占，无需锁。使用异构查找（heterogeneous lookup）避免 `string` 构造开销。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/shadowsocks/relay|relay]] | 握手时检查 salt 重放 |

## 命名空间

`psm::protocol::shadowsocks`

---

## 类: salt_pool

> 源码: `include/prism/protocol/shadowsocks/salts.hpp:27`

### 概述

Salt 重放检测池。维护已见 salt 的精确集合，配合 TTL 自动过期清理。每个 worker 线程持有独立实例（thread_local），无需线程同步。内部使用 `unordered_map` + FNV-1a 哈希 + 异构查找。

### 构造函数

```cpp
explicit salt_pool(std::int64_t ttl_seconds = 60);
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ttl_seconds` | `std::int64_t` | `60` | Salt 条目的生存时间（秒） |

---

### 函数: check_and_insert()

#### 功能说明

检查并插入 salt。如果 salt 在 TTL 内已存在则返回 `false`（重放），否则插入并返回 `true`（首次出现）。包含分摊清理逻辑：仅当距上次清理超过 1 秒时才执行过期清理。

#### 签名

```cpp
auto check_and_insert(std::span<const std::uint8_t> salt) -> bool;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `salt` | `std::span<const std::uint8_t>` | 待检查的 salt 数据 |

#### 返回值

`bool` — `true` 表示首次出现（已插入），`false` 表示重放。

#### 调用（向下）

- `cleanup()` — 分摊清理过期条目
- `entries_.find()` — 异构查找（string_view）
- `entries_.emplace()` — 插入新条目

#### 被调用（向上）

- [[protocol/shadowsocks/relay|relay]] `handshake()` 检查 salt 唯一性

#### 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] AEAD 加密上下文的 salt 重放保护
- [[protocol/shadowsocks/relay|relay]] SS2022 握手流程
- [[protocol/shadowsocks/salts|salts]] 重放检测、TTL 过期、异构查找

---

### 函数: cleanup()

#### 功能说明

清理过期的 salt 条目。遍历所有条目，移除超过 TTL 的条目。

#### 签名

```cpp
void cleanup();
```

#### 参数

无

#### 返回值

无

#### 调用（向下）

- `entries_.erase()` — 删除过期条目

#### 被调用（向上）

- `check_and_insert()` 分摊调用

#### 知识域

- [[protocol/shadowsocks/salts|salts]] TTL 过期清理、时间轮

## 相关页面

- [[protocol/shadowsocks/replay|replay]] — PacketID 重放检测
- [[protocol/shadowsocks/relay|relay]] — SS2022 中继器
- [[protocol/shadowsocks/constants|constants]] — 协议常量
