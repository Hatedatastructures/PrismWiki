---
layer: core
source: include/prism/account/
title: Account 模块总览
module: account
tags:
  - account
  - authentication
  - connection-limit
  - traffic-stats
  - write-on-copy
created: 2026-05-23
updated: 2026-05-23
---

# Account 模块总览

> 源码位置: `include/prism/account/`

## 模块职责

Account 模块负责代理服务器的用户认证与连接管理，提供以下核心能力：

- **账户存储**: 基于凭证 (credential) 的账户条目映射，支持透明查找
- **连接限制**: 每账户最大连接数控制，通过 CAS 原子操作实现无锁并发
- **流量统计**: 上行/下行字节数的原子累加，支持近似统计
- **租约管理**: RAII 风格的连接租约，确保连接异常退出时不泄漏计数
- **写时复制**: 目录更新采用 COW 策略，读取完全无锁

## 文件结构

```
include/prism/account/
├── entry.hpp        # 账户条目与租约定义
└── directory.hpp    # 账户目录（COW 无锁读取）
```

## 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| [[core/account/entry|entry]] | `entry.hpp` | 账户运行时状态 + RAII 租约 |
| [[core/account/directory|directory]] | `directory.hpp` | 凭证到条目的映射管理 |

## 架构设计

### 无锁读取模型

Account 模块采用**写时复制 (Copy-on-Write)** 策略，实现读多写少场景下的高性能并发访问：

```
┌───────────────────────────────────────────────────┐
│                  directory                        │
│                                                   │
│  entries_ptr_: atomic<shared_ptr<unordered_map>>  │
│       │                                           │
│       │  load() → shared_ptr (无锁)               │
│       ▼                                           │
│  ┌──────────────────────────────────────┐         │
│  │   unordered_map<string, shared_ptr<entry>>    │
│  │                                        │       │
│  │  "uuid-xxx" → entry{max_conn, stats}  │       │
│  │  "password-yyy" → entry{...}          │       │
│  └──────────────────────────────────────┘         │
│                                                   │
│  update_entries():                                │
│    1. 复制当前 map                                 │
│    2. 对副本执行修改                               │
│    3. CAS 原子替换 → 失败则重试                    │
└───────────────────────────────────────────────────┘
```

### 租约生命周期

```
会话创建
    │
    ▼
try_acquire(directory, credential)
    │
    ├── 查找 entry (无锁)
    │       └── 未找到 → 返回空租约 → 拒绝连接
    │
    ├── 检查连接限制
    │       └── max_connections == 0 → 无限制
    │       └── active >= max → 返回空租约 → 拒绝连接
    │
    ├── CAS 递增 active_connections
    │
    └── 返回 lease(entry_ptr)
            │
            ▼
        会话持有 lease
            │
            ├── 数据传输中:
            │     accumulate_uplink(entry, bytes)
            │     accumulate_downlink(entry, bytes)
            │
            ▼
        会话结束 (正常或异常)
            │
            ▼
        ~lease() → release()
            └── active_connections.fetch_sub(1)
```

## 透明哈希查找

directory 使用自定义的 `transparent_hash` 和 `transparent_equal`，支持以 `std::string_view` 作为查找键直接查询 `memory::string` 为键的无序映射表，避免临时字符串构造：

```cpp
// 内部键类型: memory::string (PMR 分配)
// 查询键类型: std::string_view (零拷贝)
directory dir;
dir.upsert("user-password-123");

// 以下查找不需要构造临时 memory::string
auto entry = dir.find("user-password-123");  // string_view → transparent lookup
```

## 调用链

```
main.cpp 启动
    │
    ▼
loader::build_account_directory(config)
    │
    ├── 遍历 config.authentication.users
    │       └── 对每个 user:
    │             dir.upsert(uuid_or_password, max_connections)
    │             dir.insert(other_protocol_credential, same_entry)
    │                    ↑ 多协议凭证共享同一 entry，共享连接配额
    │
    ▼
server.account_store = directory
    │
    ▼
session 创建时
    │
    ├── try_acquire(account_store, credential)
    │       → 成功: session.account_lease = lease
    │       → 失败: 拒绝连接
    │
    ├── 数据传输时
    │       → accumulate_uplink/downlink(account_lease.get(), bytes)
    │
    └── 会话结束时
            → ~lease() 自动递减 active_connections
```

## 多协议凭证共享

同一用户可以通过不同协议使用不同的凭证（如 UUID 用于 Trojan，password 用于 SOCKS5），但共享同一个 `entry` 实例和连接配额：

```cpp
// 配置加载时
auto shared_entry = std::make_shared<entry>();
shared_entry->max_connections = 10;
dir.insert("trojan-uuid-xxx", shared_entry);
dir.insert("socks5-password-yyy", shared_entry);
// 两个凭证共享 10 个连接配额
```

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[core/memory/overview|Memory]] | PMR 容器、内存池分配器 |
| 被依赖 | [[core/context/context|Context]] | session 上下文持有 lease 和 directory 指针 |
| 被依赖 | [[core/loader/overview|Loader]] | 启动时构建账户目录 |
| 被依赖 | [[core/pipeline/overview|Pipeline]] | 协议处理器使用凭证验证和租约 |

## 线程安全模型

| 操作 | 线程安全 | 机制 |
|------|----------|------|
| `directory::find()` | 是 | atomic load，无锁读取 |
| `directory::upsert()` | 是 | CAS + COW，自旋重试 |
| `directory::clear()` | 是 | CAS + COW |
| `entry` 原子字段读写 | 是 | relaxed memory order |
| `lease` 移动/析构 | 否 | 单线程使用，非线程安全 |

## 相关文档

- [[core/account/entry|entry 详解]] - 账户条目与租约的完整 API
- [[core/account/directory|directory 详解]] - 目录管理与查找接口
- [[core/context/context|Context]] - session 上下文中的账户集成
- [[core/memory/overview|Memory]] - PMR 容器与内存池
- [[core/pipeline/overview|Pipeline]] - 协议处理中的认证流程
- [[core/loader/load|Loader]] - 配置加载与账户构建
