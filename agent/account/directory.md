---
title: "directory.hpp — 账户目录管理"
source: "include/prism/agent/account/directory.hpp"
module: "agent"
type: api
tags: [agent, account, directory, 账户, 写时复制, 无锁]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[agent/account/entry|账户条目与租约]]"
  - "[[agent/context|运行时上下文]]"
  - "[[ref/memory/pmr|PMR 内存管理]]"
  - "[[ref/network/connection-pool|连接配额管理]]"
---

# directory.hpp

> 源码: `include/prism/agent/account/directory.hpp`
> 实现: `src/prism/agent/account/directory.cpp`
> 模块: [[agent|Agent]] / account

## 概述

提供基于凭证的账户条目存储和查询能力，使用写时复制（COW）策略实现无锁读取。支持透明查找，允许直接使用 `string_view` 作为查询键，避免临时字符串构造开销。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/account/entry\|entry]] | 账户条目定义 |
| 依赖 | [[memory/container\|container]] | PMR 容器类型（unordered_map） |
| 依赖 | [[memory/pool\|pool]] | 内存池分配器 |
| 被依赖 | [[agent/context\|context]] | server_context 持有 directory |
| 被依赖 | [[agent/worker/launch\|launch]] | 会话启动时验证凭证 |
| 被依赖 | [[loader/load\|loader]] | build_account_directory() 构建目录 |

## 命名空间

`psm::agent::account`

---

## 类: directory

### 概述
账户目录，管理凭证到账户条目的映射关系。采用原子共享指针实现无锁读取和写时复制更新。

### 设计意图
所有修改操作都会复制整个映射表，通过 CAS（Compare-And-Swap）原子替换，适用于读多写少的账户配置场景。内部使用自定义内存池分配器，支持与内存池资源绑定。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::allocator<std::byte>` | `allocator_` | 自定义内存分配器 |
| `std::atomic<std::shared_ptr<unordered_map>>` | `entries_ptr_` | 原子共享指针，指向当前映射表 |

### 内部辅助结构体

#### `transparent_hash`
透明哈希函数，支持 `string_view` 和 `memory::string` 的透明哈希计算。标记 `is_transparent` 使 `unordered_map::find()` 接受异构键类型。

#### `transparent_equal`
透明相等比较器，支持 `string_view` 和 `memory::string` 的混合比较，配合透明哈希实现无临时对象的键查找。

### 内部模板函数

#### `update_entries()`
```cpp
template <typename UpdateFn>
void update_entries(UpdateFn &&update_fn);
```
写时复制更新映射表。复制当前映射表，应用更新后通过 CAS 原子替换。若 CAS 失败则重试直到成功。所有写入操作（`reserve`、`upsert`、`insert`）均通过此模板实现。

---

### 构造函数

**功能说明**: 构造账户目录，初始化原子映射表指针和内存分配器。

**签名**:
```cpp
explicit directory(memory::resource_pointer resource = memory::system::thread_local_pool());
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `resource` | `memory::resource_pointer` | 内存池资源指针，默认使用线程局部池 |

**返回值**: 无（构造函数）

**调用（向下）**:
- `std::allocate_shared<unordered_map>(allocator_, 0)` 分配空映射表
- `entries_ptr_.store()` 存储初始映射表

**被调用（向上）**: [[loader/load\|loader::build_account_directory()]] 创建目录实例；[[agent/worker/worker\|worker 构造函数]] 间接持有

**涉及的知识域**: [[ref/memory/pmr\|PMR 内存管理]]

---

### reserve()

**功能说明**: 预留账户条目容量，适用于批量插入前预分配以减少写时复制的重分配次数。

**签名**:
```cpp
void reserve(std::size_t n);
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `n` | `std::size_t` | 预留的条目数量 |

**返回值**: 无

**调用（向下）**: `update_entries()` 模板（内部调用 `unordered_map::reserve(n)`）

**被调用（向上）**: [[loader/load\|loader::build_account_directory()]] 在批量注册用户前调用

**涉及的知识域**: [[ref/memory/pmr\|PMR 内存管理]]

---

### clear()

**功能说明**: 原子替换为空映射表，清空所有账户条目，原有条目在所有读取者释放后销毁。

**签名**:
```cpp
void clear();
```

**参数**: 无

**返回值**: 无

**调用（向下）**: `std::allocate_shared<unordered_map>(allocator_, 0)` 分配空映射表；`entries_ptr_.store()` 原子替换

**被调用（向上）**: 外部管理代码在需要重置账户目录时调用

**涉及的知识域**: [[ref/memory/pmr\|PMR 内存管理]]

---

### upsert()

**功能说明**: 插入或更新账户条目，若凭证不存在则创建新条目，存在则更新连接限制。操作触发写时复制。

**签名**:
```cpp
void upsert(std::string_view credential, std::uint32_t max_connections = 0);
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `credential` | `std::string_view` | 账户凭证字符串 |
| `max_connections` | `std::uint32_t` | 最大连接数限制，0 表示无限制 |

**返回值**: 无

**调用（向下）**: `update_entries()` 模板（内部通过 `operator[]` 插入/更新条目，新条目通过 `std::allocate_shared<entry>` 分配）

**被调用（向上）**: [[loader/load\|loader::build_account_directory()]] 注册用户凭证时调用（当前实现使用 `insert` 替代，`upsert` 保留供外部动态注册）

**涉及的知识域**: [[ref/network/connection-pool\|连接配额管理]]

---

### insert()

**功能说明**: 将已有 entry 以新的凭证键注册到目录中，实现多协议凭证共享同一个 entry 从而共享连接数配额。

**签名**:
```cpp
void insert(std::string_view credential, std::shared_ptr<entry> existing_entry);
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `credential` | `std::string_view` | 账户凭证字符串 |
| `existing_entry` | `std::shared_ptr<entry>` | 已有的账户条目共享指针 |

**返回值**: 无

**调用（向下）**: `update_entries()` 模板（内部通过 `operator[]` 以 `std::move` 方式插入已有条目）

**被调用（向上）**: [[loader/load\|loader::build_account_directory()]] 为每个用户的 password 和 uuid 分别调用，共享同一个 entry

**涉及的知识域**: [[ref/network/connection-pool\|连接配额管理]]

---

### find()

**功能说明**: 无锁查找账户条目，通过原子加载获取当前映射表快照后查询。

**签名**:
```cpp
[[nodiscard]] auto find(std::string_view credential) const noexcept -> std::shared_ptr<entry>;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `credential` | `std::string_view` | 账户凭证字符串 |

**返回值**: 账户条目共享指针，未找到返回 `nullptr`

**调用（向下）**:
- `entries_ptr_.load(std::memory_order_acquire)` 获取当前映射表快照
- `snapshot->find(credential)` 透明查找（利用 `transparent_hash` 和 `transparent_equal`）

**被调用（向上）**: `try_acquire()` 内部调用；[[agent/worker/launch\|launch::start()]] 间接通过 `contains()` 调用

**涉及的知识域**: [[ref/memory/pmr\|PMR 内存管理]]

---

## 自由函数: try_acquire()

**功能说明**: 尝试获取账户连接租约，通过 CAS 原子递增活跃连接数，若已达上限则返回空租约。

**签名**:
```cpp
[[nodiscard]] inline auto try_acquire(const directory &accounts, const std::string_view credential) noexcept -> lease;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `accounts` | `const directory &` | 账户目录 |
| `credential` | `std::string_view` | 账户凭证字符串 |

**返回值**: 成功返回持有租约的 `lease` 对象，失败返回空租约

**调用（向下）**:
- `directory::find()` 查找账户条目
- `entry_ptr->active_connections.compare_exchange_weak()` CAS 递增活跃连接数
- `lease(std::move(entry_ptr))` 构造租约对象

**被调用（向上）**:
- [[pipeline/protocols/vless\|pipeline::vless]] VLESS 协议处理器中验证 UUID 后获取租约
- [[pipeline/protocols/trojan\|pipeline::trojan]] Trojan 协议处理器中验证密码后获取租约
- [[protocol/http/parser\|protocol::http::authenticate_proxy_request]] HTTP 代理认证时获取租约

**涉及的知识域**: [[ref/network/connection-pool\|连接配额管理]]

---

## 自由函数: contains()

**功能说明**: 检查账户是否存在于目录中，通过 `find()` 结果判断。

**签名**:
```cpp
[[nodiscard]] inline auto contains(const directory &accounts, const std::string_view credential) noexcept -> bool;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `accounts` | `const directory &` | 账户目录 |
| `credential` | `std::string_view` | 账户凭证字符串 |

**返回值**: 存在返回 `true`，不存在返回 `false`

**调用（向下）**: `directory::find()`

**被调用（向上）**: [[agent/worker/launch\|launch::start()]] 内部的 `credential_function` lambda 中调用，用于判断凭证是否合法

**涉及的知识域**: [[ref/network/connection-pool\|连接配额管理]]

---

## 知识域

- [[ref/memory/pmr\|PMR 内存管理]]
- [[ref/network/connection-pool\|连接配额管理]]
- [[ref/programming/c++23-coroutines\|RAII 资源管理]]
