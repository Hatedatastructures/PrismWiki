---
title: "entry.hpp — 账户条目状态与租约管理"
source: "include/prism/agent/account/entry.hpp"
module: "agent"
type: api
tags: [agent, account, entry, 租约, RAII, 流量统计]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[agent/account/directory|账户目录]]"
  - "[[agent/context|运行时上下文]]"
  - "[[pipeline/primitives|隧道原语]]"
  - "[[ref/network/connection-pool|连接配额管理]]"
---

# entry.hpp

> 源码: `include/prism/agent/account/entry.hpp`
> 模块: [[agent|Agent]] / account

## 概述

定义账户状态结构和租约 RAII 封装，用于跟踪单个账户的连接数、上下行流量等运行时指标。租约对象通过引用计数自动管理活跃连接数，确保连接释放时正确递减计数器。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[agent/account/directory\|directory]] | 目录存储 entry，try_acquire 创建 lease |
| 被依赖 | [[agent/context\|context]] | session_context 持有 lease |
| 被依赖 | [[pipeline/primitives\|primitives]] | 隧道转发时累加流量 |

## 命名空间

`psm::agent::account`

---

## 结构体: entry

### 概述
账户运行时状态，存储单个账户的连接限制和实时统计信息。所有统计字段使用原子操作保证线程安全。

### 设计意图
由 `directory` 统一管理生命周期，通过 `shared_ptr` 在多连接间共享。`max_connections` 为 0 表示不限制连接数。原子字段使用 `relaxed` 内存序，仅适用于近似统计场景。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `std::uint32_t` | `max_connections` | 最大连接数，0 表示无限制 |
| `std::atomic_uint64_t` | `uplink_bytes` | 上行流量（字节） |
| `std::atomic_uint64_t` | `downlink_bytes` | 下行流量（字节） |
| `std::atomic_uint32_t` | `active_connections` | 活跃连接数 |

---

## 类: lease

### 概述
账户连接租约，RAII 封装。持有 `entry` 的共享所有权，析构时自动递减活跃连接数。

### 设计意图
确保连接异常退出时不会泄漏连接计数。租约不可拷贝，仅支持移动语义，保证所有权唯一。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `std::shared_ptr<entry>` | `state_` | 账户状态指针 |

---

### 默认构造函数

**功能说明**: 构造空租约，不持有任何账户状态。

**签名**:
```cpp
lease() = default;
```

**参数**: 无

**返回值**: 无（构造函数）

**调用（向下）**: 无

**被调用（向上）**: [[agent/context\|session_context]] 默认初始化 `account_lease` 成员

**涉及的知识域**: [[ref/programming/c++23-coroutines\|RAII 资源管理]]

---

### entry 构造函数

**功能说明**: 从账户条目构造租约，接管指针所有权，不自动递增连接数（调用方需在构造前递增）。

**签名**:
```cpp
explicit lease(std::shared_ptr<entry> state) noexcept;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `state` | `std::shared_ptr<entry>` | 账户状态指针，调用方需确保已递增活跃连接数 |

**返回值**: 无（构造函数）

**调用（向下）**: `std::move(state)` 存储到 `state_`

**被调用（向上）**: [[agent/account/directory\|try_acquire()]] CAS 成功后构造租约

**涉及的知识域**: [[ref/programming/c++23-coroutines\|RAII 资源管理]]

---

### 移动构造函数

**功能说明**: 转移源租约的所有权，移动后源对象为空。

**签名**:
```cpp
lease(lease &&other) noexcept;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `other` | `lease &&` | 源租约对象，移动后置空 |

**返回值**: 无（构造函数）

**调用（向下）**: `std::move(other.state_)` 转移所有权

**被调用（向上）**: [[agent/account/directory\|try_acquire()]] 返回临时租约对象时触发移动

**涉及的知识域**: [[ref/programming/c++23-coroutines\|RAII 资源管理]]

---

### 移动赋值运算符

**功能说明**: 先释放当前持有的租约（递减原连接计数），再转移源租约所有权。

**签名**:
```cpp
auto operator=(lease &&other) noexcept -> lease &;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `other` | `lease &&` | 源租约对象 |

**返回值**: 当前租约引用

**调用（向下）**: `release()` 释放当前租约；`std::move(other.state_)` 转移所有权

**被调用（向上）**: 会话生命周期管理中转移租约所有权时使用

**涉及的知识域**: [[ref/programming/c++23-coroutines\|RAII 资源管理]]

---

### 析构函数

**功能说明**: 析构租约并释放连接计数，调用 `release()` 递减活跃连接数。

**签名**:
```cpp
~lease();
```

**参数**: 无

**返回值**: 无（析构函数）

**调用（向下）**: `release()` 私有方法

**被调用（向上）**: [[agent/context\|session_context]] 销毁时 `account_lease` 析构

**涉及的知识域**: [[ref/programming/c++23-coroutines\|RAII 资源管理]]

---

### get()

**功能说明**: 获取底层账户状态裸指针，租约为空时返回 nullptr。

**签名**:
```cpp
[[nodiscard]] auto get() const noexcept -> entry *;
```

**参数**: 无

**返回值**: `entry *` 账户条目裸指针，空租约返回 `nullptr`

**调用（向下）**: `state_.get()`

**被调用（向上）**: [[pipeline/primitives\|tunnel]] 通过 `session_context.account_lease.get()` 获取 entry 指针用于流量累加

**涉及的知识域**: [[ref/programming/c++23-coroutines\|RAII 资源管理]]

---

### operator bool()

**功能说明**: 检查租约是否有效，通过检查内部 shared_ptr 是否非空判断。

**签名**:
```cpp
[[nodiscard]] explicit operator bool() const noexcept;
```

**参数**: 无

**返回值**: 持有有效账户状态时返回 `true`

**调用（向下）**: `static_cast<bool>(state_)`

**被调用（向上）**: 调用方在使用租约前检查有效性

**涉及的知识域**: [[ref/programming/c++23-coroutines\|RAII 资源管理]]

---

### release()（私有）

**功能说明**: 释放租约并递减活跃连接数，使用 relaxed 内存序。空租约调用此方法为空操作。

**签名**:
```cpp
void release() noexcept;
```

**参数**: 无

**返回值**: 无

**调用（向下）**: `state_->active_connections.fetch_sub(1, std::memory_order_relaxed)`；`state_.reset()` 释放 shared_ptr

**被调用（向上）**: 析构函数和移动赋值运算符内部调用

**涉及的知识域**: [[ref/network/connection-pool\|连接配额管理]]

---

## 自由函数: accumulate_uplink()

**功能说明**: 累加上行流量，使用 relaxed 内存序原子递增。空指针时直接返回。

**签名**:
```cpp
inline void accumulate_uplink(entry *state, const std::uint64_t bytes) noexcept;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `state` | `entry *` | 账户状态指针，可为空 |
| `bytes` | `const std::uint64_t` | 待累加的字节数 |

**返回值**: 无

**调用（向下）**: `state->uplink_bytes.fetch_add(bytes, std::memory_order_relaxed)`

**被调用（向上）**: [[pipeline/primitives\|tunnel]] 隧道转发中上行数据传输完成后累加

**涉及的知识域**: [[ref/network/connection-pool\|流量统计]]

---

## 自由函数: accumulate_downlink()

**功能说明**: 累加下行流量，使用 relaxed 内存序原子递增。空指针时直接返回。

**签名**:
```cpp
inline void accumulate_downlink(entry *state, const std::uint64_t bytes) noexcept;
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `state` | `entry *` | 账户状态指针，可为空 |
| `bytes` | `const std::uint64_t` | 待累加的字节数 |

**返回值**: 无

**调用（向下）**: `state->downlink_bytes.fetch_add(bytes, std::memory_order_relaxed)`

**被调用（向上）**: [[pipeline/primitives\|tunnel]] 隧道转发中下行数据传输完成后累加

**涉及的知识域**: [[ref/network/connection-pool\|流量统计]]

---

## 知识域

- [[ref/network/connection-pool\|连接配额管理与流量统计]]
- [[ref/programming/c++23-coroutines\|RAII 资源管理]]
- [[ref/memory/pmr\|PMR 内存管理]]
