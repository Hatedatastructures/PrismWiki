---
title: "core — 多路复用核心抽象基类"
source: "include/prism/multiplex/core.hpp"
implementation: "src/prism/multiplex/core.cpp"
module: "multiplex"
type: api
tags: [multiplex, core, 抽象基类, 流管理]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/smux/craft|smux::craft]]"
  - "[[multiplex/yamux/craft|yamux::craft]]"
  - "[[multiplex/duct|duct]]"
  - "[[multiplex/parcel|parcel]]"
  - "[[multiplex/config|config]]"
  - "[[multiplex/bootstrap|bootstrap]]"
  - "[[channel/transport/transmission|transmission]]"
  - "[[resolve/router|router]]"
---

# core.hpp

> 源码: `include/prism/multiplex/core.hpp`
> 实现: `src/prism/multiplex/core.cpp`
> 模块: [[multiplex/config|multiplex]]

## 概述

多路复用核心抽象基类。提供所有多路复用协议共享的会话生命周期管理、流状态跟踪和发送串行化。协议特定的帧格式、解析和协商由子类实现（如 [[multiplex/smux/craft|smux::craft]]、[[multiplex/yamux/craft|yamux::craft]]）。

core 管理三种流状态：
- **pending** — SYN 后等待地址数据
- **duct** — TCP 流双向转发（[[multiplex/duct|duct]]）
- **parcel** — UDP 数据报中继（[[multiplex/parcel|parcel]]）

设计原则：core 是协议无关的抽象层，所有帧编解码委托给子类。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[channel/transport/transmission|transmission]] | 使用 shared_transmission |
| 依赖 | [[resolve/router|router]] | 路由器引用，解析地址并连接目标 |
| 依赖 | [[memory/container|container]] | PMR 容器（unordered_map, vector） |
| 依赖 | [[multiplex/config|config]] | 多路复用配置 |
| 被依赖 | [[multiplex/smux/craft|smux::craft]] | 继承 core，实现 smux 协议 |
| 被依赖 | [[multiplex/yamux/craft|yamux::craft]] | 继承 core，实现 yamux 协议 |
| 被依赖 | [[multiplex/duct|duct]] | friend，访问 pending_/ducts_ |
| 被依赖 | [[multiplex/parcel|parcel]] | friend，访问 pending_/parcels_ |
| 被依赖 | [[multiplex/bootstrap|bootstrap]] | 返回 shared_ptr<core> |

## 命名空间

`psm::multiplex`

## 类: core

> 源码: `include/prism/multiplex/core.hpp:53`
> 实现: `src/prism/multiplex/core.cpp:14`

### 概述

管理流生命周期和发送串行化。SYN 创建 pending_entry，协议子类解析地址后连接目标，创建 duct/parcel 进行双向转发。继承 `std::enable_shared_from_this`，支持协程上下文中安全的共享指针管理。

### 类层次

```
std::enable_shared_from_this<core>
  └── core (抽象基类)
        ├── smux::craft (smux 实现)
        └── yamux::craft (yamux 实现)
```

---

### core::core (构造函数)

**功能说明**

构造 core 实例，初始化传输层连接、路由器引用、配置和 PMR 内存资源。会话处于未启动状态，需调用 `start()` 后才进入协议主循环。

**签名**

```cpp
core(channel::transport::shared_transmission transport, resolve::router &router,
     const config &cfg, memory::resource_pointer mr = {});
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `shared_transmission` | 已建立的传输层连接（通常是 TLS 隧道） |
| `router` | `router&` | 路由器引用，用于解析地址并连接目标 |
| `cfg` | `const config&` | 多路复用配置参数 |
| `mr` | `resource_pointer` | PMR 内存资源，为空时使用默认资源 |

**返回值**

构造函数，无返回值。

**调用（向下）**

- `memory::current_resource()` — 获取当前 PMR 资源

**被调用（向上）**

- `smux::craft::craft()` — smux 子类构造函数调用基类
- `yamux::craft::craft()` — yamux 子类构造函数调用基类

**知识域**

多路复用会话初始化、PMR 内存策略、shared_from_this 生命周期管理。

---

### core::start

**功能说明**

启动 mux 会话。通过 `co_spawn` 在 transport executor 上启动 `run()` 协程。异常或正常退出时自动调用 `close()` 清理资源。

**签名**

```cpp
void start();
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `core::run()` — 通过 co_spawn 启动协议主循环（纯虚，由子类实现）
- `core::close()` — 异常退出时自动调用

**被调用（向上）**

- `session` 层 — 在 mux 会话创建后调用 start

**知识域**

协程生命周期管理、co_spawn 异常处理、shared_from_this 保活。

---

### core::close

**功能说明**

关闭会话（幂等）。原子地标记非活跃，取消并关闭 transport，清空 pending_，std::move 取出 ducts_ 和 parcels_ 后逐一 close。多次调用无副作用。

**签名**

```cpp
virtual void close();
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `transport_->cancel()` — 取消传输层所有挂起操作
- `transport_->close()` — 关闭传输层连接
- `duct::close()` — 关闭每个活跃 TCP 管道
- `parcel::close()` — 关闭每个活跃 UDP 管道

**被调用（向上）**

- `core::start()` — 异常退出回调中调用
- `core::~core()` — 析构函数中调用
- `yamux::craft::close()` — yamux 子类 override 后调用基类
- `smux::craft` 帧循环退出后 — 通过 run() 的异常处理

**知识域**

幂等关闭模式、std::move 避免 iterator invalidation、原子交换保证单次执行。

---

### core::is_active

**功能说明**

检查会话是否活跃。通过原子变量读取 active_ 标志，使用 acquire 内存序保证可见性。

**签名**

```cpp
[[nodiscard]] bool is_active() const noexcept;
```

**参数**

无。

**返回值**

`bool` — true 表示会话正在运行。

**调用（向下）**

无。

**被调用（向上）**

- `duct::target_read_loop()` — 检查 mux 会话是否仍活跃
- `smux::craft::send_loop()` — 检查会话状态
- `yamux::craft::send_loop()` — 检查会话状态
- `yamux::craft::send_data()` — 窗口等待期间检查会话状态
- `yamux::craft::ping_loop()` — 检查会话状态

**知识域**

原子内存序、acquire/release 语义。

---

### core::send_data (纯虚)

**功能说明**

发送数据帧到客户端。将 payload 编码为协议数据帧并发送。payload 通过 move 传递，不执行额外拷贝。由 duct::target_read_loop 和 parcel::relay_datagram 调用。

**签名**

```cpp
virtual auto send_data(std::uint32_t stream_id, memory::vector<std::byte> payload) const
    -> net::awaitable<void> = 0;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 目标流标识符 |
| `payload` | `memory::vector<byte>` | 要发送的数据（所有权转移） |

**返回值**

`net::awaitable<void>` — 协程，完成时帧已推入发送通道。

**调用（向下）**

- 子类实现：`smux::craft::push_frame()` 或 `yamux::craft::push_frame()`

**被调用（向上）**

- `duct::target_read_loop()` — target 读到数据后发送到 mux 客户端
- `parcel::downlink_loop()` — UDP 响应编码后回传
- `smux::craft::activate_stream()` — 发送成功/错误状态
- `yamux::craft::activate_stream()` — 发送成功/错误状态
- `yamux::craft::dispatch_data()` — pending 流中更新接收窗口

**知识域**

零拷贝传递、所有权转移语义、协程异步发送。

---

### core::send_fin (纯虚)

**功能说明**

发送 FIN 帧到客户端。异步发送半关闭帧，不阻塞调用者。由 duct::target_read_loop 在 target EOF 后调用，或由 activate_stream 在连接失败时调用。

**签名**

```cpp
virtual void send_fin(std::uint32_t stream_id) = 0;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 目标流标识符 |

**返回值**

`void` — 异步发送，立即返回。

**调用（向下）**

- 子类实现：通过 co_spawn 异步发送 FIN 帧

**被调用（向上）**

- `duct::target_read_loop()` — target EOF 后通知 mux 端半关闭
- `smux::craft::activate_stream()` — 地址解析失败或连接失败时
- `yamux::craft::activate_stream()` — 地址解析失败或连接失败时

**知识域**

半关闭语义、co_spawn 异步发送、不阻塞调用者模式。

---

### core::executor (纯虚)

**功能说明**

获取当前 executor，用于 duct/parcel 协程调度。

**签名**

```cpp
[[nodiscard]] virtual net::any_io_executor executor() const = 0;
```

**参数**

无。

**返回值**

`net::any_io_executor` — transport 的执行器。

**调用（向下）**

- 无（返回值供调用者使用）

**被调用（向上）**

- `duct` 构造函数 — 缓存 executor
- `parcel` 构造函数 — 缓存 executor
- `smux::craft::send_fin()` — co_spawn 目标 executor
- `yamux::craft::send_fin()` — co_spawn 目标 executor

**知识域**

Boost.Asio executor 模型、协程调度。

---

### core::run (纯虚，私有)

**功能说明**

协议主循环，由子类实现。由 start() 通过 co_spawn 启动，退出时自动 close()。

**签名**

```cpp
virtual auto run() -> net::awaitable<void> = 0;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，协议会话主循环。

**调用（向下）**

- 子类实现各自的帧读取和分发逻辑

**被调用（向上）**

- `core::start()` — 通过 co_spawn 启动

**知识域**

模板方法模式、协程主循环。

---

### core::remove_duct (受保护，虚)

**功能说明**

从活跃管道映射中移除指定 TCP 管道。由 duct::close() 调用，子类可 override 清理协议特定资源（如 yamux 窗口）。

**签名**

```cpp
virtual void remove_duct(std::uint32_t stream_id);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 要移除的流标识符 |

**返回值**

`void`

**调用（向下）**

- `ducts_.erase()` — 从映射中移除

**被调用（向上）**

- `duct::close()` — 关闭时从 owner 移除自身
- `yamux::craft::remove_duct()` — override 后调用基类

**知识域**

friend 访问权限、虚函数 override 模式。

---

### core::remove_parcel (受保护，虚)

**功能说明**

从活跃管道映射中移除指定 UDP 管道。由 parcel::close() 调用，子类可 override 清理协议特定资源（如 yamux 窗口）。

**签名**

```cpp
virtual void remove_parcel(std::uint32_t stream_id);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 要移除的流标识符 |

**返回值**

`void`

**调用（向下）**

- `parcels_.erase()` — 从映射中移除

**被调用（向上）**

- `parcel::close()` — 关闭时从 owner 移除自身
- `yamux::craft::remove_parcel()` — override 后调用基类

**知识域**

friend 访问权限、虚函数 override 模式。

---

## 内部结构: pending_entry

| 字段 | 类型 | 说明 |
|------|------|------|
| `buffer` | `memory::vector<byte>` | 累积的地址+数据 |
| `connecting` | `bool` | 是否已发起连接 |

## 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `transport_` | `shared_transmission` | 底层传输连接 |
| `router_` | `router&` | 路由器引用 |
| `config_` | `const config&` | mux 配置 |
| `mr_` | `resource_pointer` | PMR 内存资源 |
| `active_` | `atomic<bool>` | 会话活跃标志 |
| `pending_` | `unordered_map<uint32, pending_entry>` | 待连接流 |
| `ducts_` | `unordered_map<uint32, shared_ptr<duct>>` | 活跃 TCP 管道 |
| `parcels_` | `unordered_map<uint32, shared_ptr<parcel>>` | 活跃 UDP 管道 |

## 设计约束

- 线程安全：单个实例非线程安全，应在 transport executor 上串行使用
- 发送操作通过子类的串行化机制确保帧不会被交错写入
- close() 是幂等操作，多次调用无副作用
- 子类必须实现 send_data、send_fin、executor 和 run 四个纯虚函数

## 调用链

```
session → core::start() → co_spawn(run())
  run() → frame_loop() → handle_syn/handle_push/handle_fin
  activate_stream() → router → duct/parcel
  duct::target_read_loop() → core::send_data() → transport
```
