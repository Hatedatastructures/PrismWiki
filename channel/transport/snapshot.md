---
title: "snapshot — 可回滚的传输层包装器"
source: "include/prism/channel/transport/snapshot.hpp"
module: "channel"
type: api
tags: [channel, transport, snapshot, 回滚, 预读, 伪装]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/transport/transmission
  - stealth/scheme
  - memory/container
  - ref/programming/c++23-coroutines
  - ref/memory/pmr
  - ref/network/tcp
---

# snapshot.hpp

> 源码: `include/prism/channel/transport/snapshot.hpp`
> 模块: [[channel|channel]]

## 概述

可回滚的传输层包装器。自动捕获所有从内层传输读取的字节，支持 `rewind()` 回到起点重新读取。

用途：TLS 伪装方案的依次尝试 — 每个 scheme 读取的数据被 snapshot 捕获，失败时 rewind，下一个 scheme 从同一起点重试。参见 [[stealth/scheme|stealth::scheme]]。

设计约束：
- `rewind()` 仅在未发生写入时有效（`wrote_ == false`）
- 认证阶段是纯读取，安全 rewind
- 一旦开始写入（如 TLS 握手），transport 状态不可恢复

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[memory/container|memory]] | PMR 向量用于 captured_ 缓冲区 |
| 被依赖 | [[stealth/scheme|scheme]] | 伪装方案依次尝试 |

## 命名空间

`psm::channel::transport`

## 类: snapshot

> 源码: `include/prism/channel/transport/snapshot.hpp:40`

### 构造函数

---

#### snapshot(inner, mr)

**功能说明**：构造 snapshot 包装器，包装内层传输并使用 PMR 内存资源分配捕获缓冲区。

**签名**：
```cpp
explicit snapshot(shared_transmission inner, memory::resource_pointer mr = memory::current_resource());
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `inner` | `shared_transmission` | 被包装的内层传输 |
| `mr` | `memory::resource_pointer` | [[ref/memory/pmr|PMR]] 内存资源，用于 captured_ 缓冲区分配 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`make_snapshot()` 工厂函数

**涉及的知识域**：[[ref/memory/pmr|PMR]]

### 成员函数

---

#### is_reliable()

**功能说明**：委托给内层传输，检查是否可靠。

**签名**：
```cpp
[[nodiscard]] bool is_reliable() const noexcept override;
```

**参数**：无

**返回值**：`bool` — 内层传输的可靠性

**调用（向下）**：`inner_->is_reliable()`

**被调用（向上）**：协议层根据可靠性选择策略

**涉及的知识域**：[[ref/network/tcp|TCP]], [[ref/network/udp|UDP]]

---

#### executor()

**功能说明**：委托给内层传输的执行器。内层传输为空时抛出异常。

**签名**：
```cpp
[[nodiscard]] executor_type executor() const override;
```

**参数**：无

**返回值**：`executor_type` — 内层传输的执行器

**调用（向下）**：`inner_->executor()`

**被调用（向上）**：[[ref/programming/boost-asio|Boost.Asio]] 框架

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### async_read_some()

**功能说明**：两阶段读取：Phase 1 从捕获缓冲区回放已读数据，Phase 2 从内层传输读取新数据并追加到捕获缓冲区。

**签名**：
```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<std::byte>` | 接收缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 读取的字节数

**调用（向下）**：`inner_->async_read_some()`（Phase 2）

**被调用（向上）**：[[stealth/scheme|scheme]] 依次尝试时读取 ClientHello

**涉及的知识域**：[[ref/programming/c++23-coroutines|C++23 协程]]

**两阶段逻辑**：
1. **Phase 1（回放）** — 若 `read_pos_ < captured_.size()`，从 `captured_` 复制到 buffer（同步，无异步开销）
2. **Phase 2（内层读取）** — 从内层传输异步读取，同时追加到 `captured_`

---

#### async_write_some()

**功能说明**：标记已写入（禁止 rewind），直接委托给内层传输。

**签名**：
```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 发送缓冲区 |
| `ec` | `std::error_code &` | 错误码输出参数 |

**返回值**：`net::awaitable<std::size_t>` — 写入的字节数

**调用（向下）**：`inner_->async_write_some()`

**被调用（向上）**：协议握手完成后写入响应

**涉及的知识域**：[[ref/programming/c++23-coroutines|C++23 协程]]

---

#### close()

**功能说明**：委托给内层传输的 `close()`。

**签名**：
```cpp
void close() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`inner_->close()`

**被调用（向上）**：会话结束时调用

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### cancel()

**功能说明**：委托给内层传输的 `cancel()`。

**签名**：
```cpp
void cancel() override;
```

**参数**：无

**返回值**：无

**调用（向下）**：`inner_->cancel()`

**被调用（向上）**：超时处理或会话中断

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### rewind()

**功能说明**：将读取位置归零，下次 `async_read_some` 将从 `captured_` 起点回放。不清空捕获数据。调用前应检查 `can_rewind()`。

**签名**：
```cpp
void rewind() noexcept;
```

**参数**：无

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：[[stealth/scheme|scheme]] 执行失败后回滚到起点，下一个 scheme 重试

**涉及的知识域**：[[ref/anti-censorship/tls-fingerprint|TLS 指纹伪装]]

---

#### can_rewind()

**功能说明**：检查是否可以回滚。仅在未发生写入时可回滚，一旦写入过 transport 状态不可恢复。

**签名**：
```cpp
[[nodiscard]] bool can_rewind() const noexcept;
```

**参数**：无

**返回值**：`bool` — `true` 表示可回滚（未写入过数据）

**调用（向下）**：无

**被调用（向上）**：[[stealth/scheme|scheme]] 调用 `rewind()` 前检查

**涉及的知识域**：[[ref/anti-censorship/tls-fingerprint|TLS 指纹伪装]]

---

#### inner()

**功能说明**：获取内层传输的共享指针。

**签名**：
```cpp
[[nodiscard]] auto inner() const noexcept -> shared_transmission;
```

**参数**：无

**返回值**：`shared_transmission` — 内层传输

**调用（向下）**：无

**被调用（向上）**：需要访问内层传输的场景

**涉及的知识域**：[[channel/transport/transmission|transmission]]

### 工厂函数

---

#### make_snapshot()

**功能说明**：创建 snapshot 包装器的工厂函数。

**签名**：
```cpp
inline auto make_snapshot(shared_transmission inner, memory::resource_pointer mr = memory::current_resource())
    -> shared_transmission;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `inner` | `shared_transmission` | 被包装的内层传输 |
| `mr` | `memory::resource_pointer` | PMR 内存资源 |

**返回值**：`shared_transmission` — 包装后的 snapshot 传输

**调用（向下）**：`snapshot(inner, mr)` 构造函数

**被调用（向上）**：[[stealth/scheme|scheme]] 创建可回滚传输层

**涉及的知识域**：[[ref/memory/pmr|PMR]]

### 使用流程

```
make_snapshot(inner) → snapshot 构造
  → scheme_1 尝试 → async_read_some 读取并捕获
  → scheme_1 失败 → can_rewind() == true → rewind()
  → scheme_2 尝试 → async_read_some 从 captured_ 回放
  → scheme_2 成功 → async_write_some → wrote_ = true
  → 后续 rewind() 无效
```
