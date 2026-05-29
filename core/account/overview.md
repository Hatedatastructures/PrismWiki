---
layer: core
module: account
source: include/prism/account/
title: Account 模块总览
tags:
  - account
  - authentication
  - connection-limit
  - traffic-stats
  - write-on-copy
created: 2026-05-23
updated: 2026-05-27
---

# Account 模块总览

Account 模块负责代理服务器的用户认证与连接管理：账户存储、连接限制（CAS 无锁）、流量统计（原子累加）、RAII 租约、COW 目录。

## 设计决策

### 为什么用 COW (Copy-on-Write) 目录？

`directory` 的读操作（每次认证查找）是热路径，写操作（配置热加载更新）极少。COW 让读取端通过 `atomic load` 获取 `shared_ptr`，完全无锁。写入端复制整个 map 修改后 CAS 替换，不影响进行中的读取。

**后果**: 更新操作需要复制整个 map，O(n) 内存开销。但更新频率极低（秒级/分钟级），可接受。

### 为什么 lease 是 RAII 而非手动 release？

连接异常退出（RST、超时、协程取消）时，如果忘记调用 `release()`，`active_connections` 永远不递减，最终达到上限拒绝所有新连接。RAII 析构函数保证无论退出路径如何都递减计数。

**后果**: lease 不可拷贝（防止重复递减），只可移动。

## 文件结构

```
include/prism/account/
├── entry.hpp        # 账户条目与租约定义
└── directory.hpp    # 账户目录（COW 无锁读取）
```

## 核心组件

| 组件 | 文件 | 职责 |
|------|------|------|
| [[core/account/entry\|entry]] | `entry.hpp` | 账户运行时状态（原子计数器）+ RAII 租约 |
| [[core/account/directory\|directory]] | `directory.hpp` | 凭证到条目的 COW 映射，透明哈希查找 |

## 约束

### lease 必须在同一线程析构

**类型**: 线程安全

**规则**: `lease` 的析构函数调用 `active_connections.fetch_sub(1)`，虽然原子操作本身线程安全，但 lease 对象不可跨线程传递。

**违反后果**: 重复递减或漏递减。

**源码依据**: `entry.hpp` lease 析构函数

### directory 更新有自旋重试

**类型**: 性能

**规则**: `upsert()` 使用 CAS 替换 `shared_ptr`，如果并发写入导致 CAS 失败，会自旋重试（复制 + CAS）。

**违反后果**: 高频并发写入场景（不应发生）下性能退化。

## 线程安全模型

| 操作 | 线程安全 | 机制 |
|------|----------|------|
| `directory::find()` | 是 | atomic load，无锁 |
| `directory::upsert()` | 是 | CAS + COW，自旋重试 |
| `entry` 原子字段读写 | 是 | relaxed memory order |
| `lease` 移动/析构 | 否 | 单线程使用 |

## 引用关系

### 依赖

- [[core/memory/overview|Memory]]：PMR 容器

### 被引用

- [[core/context/context|Context]]：session 持有 lease 和 directory 指针
- [[core/loader/overview|Loader]]：启动时构建账户目录
- [[core/pipeline/overview|Pipeline]]：协议处理器使用凭证验证

## 故障场景

### 1. CAS 自旋活锁

**触发**: 多线程高频并发调用 `directory::upsert()`，每次 CAS 失败后复制整个 map 重试。

**表现**: CPU 飙升，更新延迟线性增长。正常场景下配置热加载频率极低（秒/分钟级），不会触发。

**恢复**: 限流 upsert 调用频率，或将账户更新收敛到单线程执行。

### 2. lease 跨线程移动导致重复递减

**触发**: 在线程 A 创建 lease，移动到线程 B，两个线程各自析构。

**表现**: `active_connections` 被递减两次，计数器变为负数或溢出，后续连接限制失效（永远允许或永远拒绝）。

**恢复**: 无法自动恢复。代码审计需确保 lease 不跨线程传递。

### 3. entry 指针悬垂

**触发**: `directory::clear()` 或 `upsert()` 替换 map 后，旧 map 中的 entry 被销毁，但某个 session 仍持有旧 `shared_ptr<entry>`。

**表现**: 因 `shared_ptr` 引用计数保护，实际不会悬垂。但如果 `shared_ptr` 被错误 reset（如手动管理生命周期），则访问已销毁 entry 导致 UB。

**恢复**: `shared_ptr` 语义保证安全。禁止手动 reset entry 指针。

### 4. try_acquire 连接数溢出

**触发**: `active_connections` 使用 `relaxed` 内存序的 CAS，极端高并发下多个线程同时通过 CAS 检查。

**表现**: 实际连接数可能短暂超过 `max_connections` 1-2 个（relaxed 无全局序保证）。

**恢复**: 对于代理服务器的连接限制场景，短暂溢出可接受。若需严格限制，需使用 `seq_cst` 序，但会降低性能。

### 5. directory 未初始化即查找

**触发**: 在 `loader::build_account_directory()` 完成前，session 尝试认证查找。

**表现**: `find()` 返回 `nullptr`，所有认证失败，连接被拒绝。

**恢复**: 启动顺序保证 directory 先于 listener 构建。违反启动顺序是编程错误。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| directory 构建顺序 | loader → account | `loader::build_account_directory()` 在 listener 启动前完成构建 |
| 认证查找 | pipeline → account | 协议处理器（Trojan/VLESS/SS2022）调用 `try_acquire()` 或 `contains()` 验证凭证 |
| 流量统计 | tunnel → account | 双向转发完成后调用 `accumulate_uplink()`/`accumulate_downlink()` |
| lease 生命周期 | session → account | session 持有 lease，session 析构时自动释放连接计数 |
| 多凭证共享 entry | loader → account | 同一用户的多协议凭证通过 `insert(credential, existing_entry)` 共享连接配额 |
| PMR 内存 | memory → account | `directory` 和内部 map 使用 PMR 分配器，依赖 `memory::system` 初始化 |

## 变更敏感度

### 对外影响

| 变更类型 | 影响范围 | 破坏性 |
|----------|----------|--------|
| `entry` 字段变更 | 所有使用 `entry` 的 session/context/tunnel 代码需适配 | **高** |
| `lease` 语义变更（如添加拷贝） | 所有持有 lease 的 session 生命周期逻辑需审计 | **高** |
| `try_acquire` 签名变更 | 所有协议处理器的认证流程需修改 | **高** |
| `directory::find()` 返回类型变更 | 所有认证查找点需适配 | **高** |
| `max_connections` 语义变更（如 0 含义改变） | 所有连接限制逻辑需审查 | **高** |

### 对内影响

| 变更类型 | 影响范围 | 风险 |
|----------|----------|------|
| COW 策略替换（如改用读写锁） | `directory` 整体重写，读路径从无锁变为加锁 | **高** |
| 透明哈希/比较器变更 | `transparent_hash`/`transparent_equal` 影响查找正确性 | 中 |
| 原子操作内存序变更 | `relaxed` → `seq_cst` 影响性能，反向可能引入竞态 | 中 |
| PMR 分配器替换 | `directory` 构造函数和内部 map 的分配策略需同步修改 | 低 |
