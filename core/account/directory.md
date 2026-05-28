---
layer: core
source: include/prism/account/directory.hpp
title: Account Directory
module: account
tags:
  - directory
  - COW
  - transparent-hash
  - lock-free-read
  - CAS
created: 2026-05-23
updated: 2026-05-23
---

# Account Directory

> 源码位置: `include/prism/account/directory.hpp`

## 概述

`directory` 管理凭证 (credential) 到账户条目的映射关系，采用原子共享指针实现**无锁读取**和**写时复制 (Copy-on-Write) 更新**。是 [[core/account/overview|Account 模块]] 的核心存储组件。

## 核心接口

| 方法 | 说明 |
|------|------|
| `directory(resource)` | 构造，默认使用 `local_pool` |
| `reserve(n)` | 预分配容量，触发 COW |
| `clear()` | 原子替换为空表 |
| `upsert(credential, max)` | 插入或更新，COW + CAS |
| `insert(credential, entry)` | 注册已有 entry 到新凭证键，实现多协议凭证共享配额 |
| `find(credential)` | 无锁读取，返回 `shared_ptr<entry>` |

## 设计决策

### 为什么用 COW 而不是读写锁？

每次认证查找（`find()`）是热路径，每个连接至少调用一次。读写锁即使在读模式下也需要原子操作（引用计数或序列号），开销约 10-20ns。COW 通过 `atomic load` 获取 `shared_ptr` 后直接查询，完全无锁，开销约 2-5ns。写入端（配置热加载）频率极低（秒级/分钟级），复制整个 map 的 O(n) 开销可忽略。

**后果**: 写操作需要复制整个 map，O(n) 内存开销。但更新频率极低，可接受。

### 为什么用透明哈希？

协议解析产生的凭证是 `string_view`（零拷贝视图），而 map 的键类型是 `memory::string`。普通哈希表查找需要构造临时 `memory::string`，触发 PMR 分配。`transparent_hash` + `transparent_equal` 通过 `is_transparent` 标记启用 C++20 异构查找，`string_view` 直接查表，热路径零分配。

**后果**: 两个函子必须保证跨类型哈希一致性（统一用 `std::hash<string_view>` 计算哈希值）。

### 为什么 insert() 支持注册已有 entry？

Trojan 的 UUID 凭证和 SOCKS5 的密码凭证可能属于同一用户。`insert()` 让多个凭证键指向同一个 `entry`，共享连接数配额。否则同一用户用不同协议连接时，连接数限制会被绕过。

**后果**: `entry` 的 `shared_ptr` 被多个 map key 引用，`entry` 生命周期直到所有 key 被移除且所有 lease 释放。

### 为什么 try_acquire() 用 CAS 循环而非 fetch_add + 条件检查？

`fetch_add` 先递增再检查的方式在高并发下会导致瞬时超出 `max_connections`：多个线程同时递增后才发现超限。CAS 循环先读当前值判断是否超限，再尝试原子替换，保证严格不超限。

**后果**: 高并发满载时 CAS 自旋重试次数增加，但实际场景中连接建立频率远低于连接数上限。

## 约束

### 写操作有自旋重试

**类型**: 性能

**规则**: `upsert()`/`insert()`/`clear()` 通过 CAS 替换 `shared_ptr`，并发写入导致 CAS 失败时自旋重试（复制 + CAS）。

**违反后果**: 高频并发写入场景（不应发生）下性能退化。

**源码依据**: `directory.hpp:160-169` update_entries 循环

### directory 构造应在单线程完成

**类型**: 调用顺序

**规则**: 账户目录应在启动阶段（loader）单线程构建，构建完成后通过 `shared_ptr` 传递给 worker 线程只读访问。

**违反后果**: 运行期间写入（热加载除外）会触发 COW 复制，高频写入导致性能退化和内存抖动。

**源码依据**: `directory.hpp:96` 构造函数默认 `local_pool`

## 引用关系

### 依赖

- [[core/account/entry|entry]]：目录管理的账户条目
- [[core/memory/overview|Memory]]：PMR 容器与内存池

### 被引用

- [[core/context/context|Context]]：session 持有 directory 指针
- [[core/loader/overview|Loader]]：启动时构建账户目录
