---
layer: core
module: account
source: I:/code/Prism/include/prism/account/
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
