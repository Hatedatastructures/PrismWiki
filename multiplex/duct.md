---
title: "duct — 多路复用 TCP 流管道"
source: "include/prism/multiplex/duct.hpp"
implementation: "src/prism/multiplex/duct.cpp"
module: "multiplex"
type: api
tags: [multiplex, duct, tcp, 流管道, 双向转发]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/core|core]]"
  - "[[multiplex/parcel|parcel]]"
  - "[[channel/transport/transmission|transmission]]"
  - "[[memory/container|container]]"
  - "[[multiplex/smux/craft|smux::craft]]"
  - "[[multiplex/yamux/craft|yamux::craft]]"
---

# duct.hpp

> 源码: `include/prism/multiplex/duct.hpp`
> 实现: `src/prism/multiplex/duct.cpp`
> 模块: [[multiplex/core|multiplex]]

## 概述

多路复用 TCP 流管道。协议无关的双向 TCP 转发管道，每条 duct 绑定一个已连接的 target 传输层，提供 mux 帧到 target 的透明双向转发。

- **target_read_loop**（target → mux，客户端下载）：独立协程循环读取 target 数据，通过 [[multiplex/core|core::send_data]] 发送到 mux 客户端
- **on_mux_data**（mux → target，客户端上传）：接收 mux 帧数据，推入有界 `write_channel_`，由 `target_write_loop` 写入 target

有界 `write_channel_` 提供反压，防止快生产者淹没慢 target。设计原则：duct 是协议无关的，通过 core 虚函数接口发送帧，不依赖具体协议。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[multiplex/core|core]] | 通过 weak_ptr 持有 owner，调用 send_data/send_fin/remove_duct |
| 依赖 | [[channel/transport/transmission|transmission]] | 使用 shared_transmission 作为 target |
| 依赖 | [[memory/container|container]] | PMR 容器（vector） |
| 被依赖 | [[multiplex/core|core]] | core::ducts_ 持有 shared_ptr<duct> |
| 被依赖 | [[multiplex/smux/craft|smux::craft]] | activate_stream 创建 duct |
| 被依赖 | [[multiplex/yamux/craft|yamux::craft]] | activate_stream 创建 duct |

## 命名空间

`psm::multiplex`

---

## 类: duct

> 源码: `include/prism/multiplex/duct.hpp:52`
> 实现: `src/prism/multiplex/duct.cpp:19`

### 概述

管理单条 TCP 流的完整生命周期，从 `activate_stream` 创建 target 连接成功后开始，到任一端关闭或 mux 会话结束时终止。

半关闭语义：mux 端收到 FIN 时标记 `mux_closed_` 并关闭 `write_channel_`；target 端读到 EOF 时标记 `target_closed_` 并调用 `owner_->send_fin`。两端均关闭后 duct 自行析构。

### 类层次

```
std::enable_shared_from_this<duct>
  └── duct
```

---

### duct::duct (构造函数)

**功能说明**

构造 duct 实例。初始化流标识符、owner 弱引用、target 传输层和 PMR 内存资源。`read_size_` 根据 buffer_size 和 max_frame_payload (65535) 取较小值，确保 target 读取的单次数据量不超过 mux 帧最大载荷。

**签名**

```cpp
duct(std::uint32_t stream_id, std::shared_ptr<core> owner,
     channel::transport::shared_transmission target,
     std::uint32_t buffer_size, memory::resource_pointer mr);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符，由 mux 协议在 SYN 帧中分配 |
| `owner` | `shared_ptr<core>` | 所属 core 的共享指针 |
| `target` | `shared_transmission` | 已连接的目标传输层，生命周期转移给 duct |
| `buffer_size` | `uint32_t` | 每流读取缓冲区大小 |
| `mr` | `resource_pointer` | PMR 内存资源 |

**返回值**

构造函数，无返回值。

**调用（向下）**

- `std::min()` — 计算 read_size_ 上限

**被调用（向上）**

- `make_duct()` — 工厂函数调用构造函数
- `smux::craft::activate_stream()` — TCP 连接成功后创建 duct
- `yamux::craft::activate_stream()` — TCP 连接成功后创建 duct

**知识域**

帧载荷上限约束、PMR 内存分配、所有权转移。

---

### duct::start

**功能说明**

启动 target 读循环和写循环。通过 co_spawn 在 target executor 上启动两个独立协程：target_read_loop（target → mux，客户端下载）和 target_write_loop（mux → target，客户端上传）。两个协程通过 shared_from_this 持有 self 保活。

**签名**

```cpp
void start();
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `co_spawn(target_read_loop())` — 启动下载方向协程
- `co_spawn(target_write_loop())` — 启动上传方向协程

**被调用（向上）**

- `smux::craft::activate_stream()` — 创建 duct 后启动
- `yamux::craft::activate_stream()` — 创建 duct 后启动

**知识域**

co_spawn 协程启动、shared_from_this 保活、异常回调关闭模式。

---

### duct::on_mux_data

**功能说明**

接收 mux 数据并写入 target。将数据推入有界 write_channel_，由独立的 target_write_loop 写入 target。有界通道提供反压：当通道满时协程挂起等待，防止帧循环无限制推送数据导致内存膨胀。由 craft::dispatch_push 通过 co_spawn 非阻塞调用，不阻塞帧循环。

**签名**

```cpp
auto on_mux_data(memory::vector<std::byte> data) -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `memory::vector<byte>` | 来自 mux 数据帧的数据（所有权转移） |

**返回值**

`net::awaitable<void>` — 协程，数据推入通道后完成。

**调用（向下）**

- `write_channel_.async_send()` — 推入有界写通道

**被调用（向上）**

- `smux::craft::dispatch_push()` — 通过 co_spawn 非阻塞调用
- `yamux::craft::dispatch_data()` — 通过 co_spawn 非阻塞调用

**知识域**

有界通道反压机制、所有权转移、非阻塞分发模式。

---

### duct::on_mux_fin

**功能说明**

处理 mux 端 FIN，触发半关闭。标记 mux_closed_ 为 true，shutdown target 发送方向。target 端可能仍在发送数据，target_read_loop 继续运行直到 target EOF 后调用 owner_->send_fin 完成全双工关闭。

**签名**

```cpp
void on_mux_fin();
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `target_->shutdown_write()` — shutdown target 发送方向
- `duct::close()` — 若 target 也已关闭则完全关闭

**被调用（向上）**

- `smux::craft::handle_fin()` — 收到 FIN 帧时调用
- `yamux::craft::handle_fin()` — 收到 FIN 标志时调用
- `yamux::craft::handle_rst()` — 收到 RST 标志时调用
- `yamux::craft::handle_window_update()` — 收到 FIN/RST 标志时调用

**知识域**

TCP 半关闭语义（shutdown write）、双端关闭状态机。

---

### duct::close

**功能说明**

关闭管道（幂等）。首次调用时：标记 closed_ 为 true，关闭 write_channel_ 通知 target_write_loop 退出，关闭 target 传输层，调用 owner_->remove_duct 从 core 的 ducts_ 映射中移除自身。多次调用无副作用。

**签名**

```cpp
void close();
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `write_channel_.cancel()` — 通知 target_write_loop 退出
- `target_->close()` — 关闭 target 传输层
- `owner_->remove_duct()` — 从 core 映射移除自身

**被调用（向上）**

- `duct::~duct()` — 析构函数中调用
- `duct::on_mux_fin()` — 两端均关闭时调用
- `duct::target_write_loop()` — 写入失败时调用
- `duct::target_read_loop()` — 退出回调中调用
- `core::close()` — 关闭所有 duct

**知识域**

幂等关闭模式、target 生命周期管理、防止 completion handler 悬空。

---

### duct::stream_id

**功能说明**

获取流标识符。返回 mux 协议在 SYN 帧中分配的流标识符。

**签名**

```cpp
[[nodiscard]] std::uint32_t stream_id() const noexcept;
```

**参数**

无。

**返回值**

`std::uint32_t` — mux 协议分配的流标识符。

**调用（向下）**

无。

**被调用（向上）**

- 日志输出 — 记录流标识符

**知识域**

内联 getter、noexcept 保证。

---

### duct::target_read_loop (私有)

**功能说明**

target 读循环（客户端下行/下载方向）。循环读取 target 数据，每次读取量不超过 read_size_（受 mux 帧最大载荷限制），读取后通过 owner_->send_data 将数据编码为 mux PSH 帧发送到客户端。target EOF 时标记 target_closed_，调用 owner_->send_fin 通知 mux 端半关闭。mux_closed_ 为 true 时停止发送（yamux 协议：客户端 FIN 后不再发送 WindowUpdate）。

**签名**

```cpp
auto target_read_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，循环直到任一端关闭。

**调用（向下）**

- `target_->async_read_some()` — 从 target 读取数据
- `owner_->send_data()` — 发送数据到 mux 客户端
- `owner_->send_fin()` — target EOF 后通知 mux 端
- `duct::close()` — 退出回调中关闭

**被调用（向上）**

- `duct::start()` — 通过 co_spawn 启动

**知识域**

协程循环读取、PMR vector 复用、半关闭状态检查、所有权 move 传递。

---

### duct::target_write_loop (私有)

**功能说明**

target 写循环（客户端上行/上传方向）。循环从 write_channel_ 取出数据并写入 target 传输层。write_channel_ 关闭时（on_mux_fin 或 close 触发）退出循环。写入失败时调用 close() 关闭整个管道。

**签名**

```cpp
auto target_write_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，循环直到通道关闭或写入失败。

**调用（向下）**

- `write_channel_.async_receive()` — 从通道取数据
- `target_->async_write()` — 写入 target 传输层
- `duct::close()` — 写入失败时关闭

**被调用（向上）**

- `duct::start()` — 通过 co_spawn 启动

**知识域**

有界通道消费、协程写循环、错误处理。

---

## 自由函数: make_duct

**功能说明**

创建 duct 共享指针的工厂函数。

**签名**

```cpp
[[nodiscard]] inline auto make_duct(std::uint32_t stream_id, std::shared_ptr<core> owner,
                                    channel::transport::shared_transmission target,
                                    std::uint32_t buffer_size, memory::resource_pointer mr = {})
    -> std::shared_ptr<duct>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |
| `owner` | `shared_ptr<core>` | 所属 core 的共享指针 |
| `target` | `shared_transmission` | 已连接的目标传输层 |
| `buffer_size` | `uint32_t` | 每流读取缓冲区大小 |
| `mr` | `resource_pointer` | PMR 内存资源 |

**返回值**

`std::shared_ptr<duct>` — duct 的共享指针。

**调用（向下）**

- `std::make_shared<duct>()` — 构造 duct 实例

**被调用（向上）**

- `smux::craft::activate_stream()` — TCP 连接成功后创建
- `yamux::craft::activate_stream()` — TCP 连接成功后创建

**知识域**

工厂函数模式、inline 头文件定义。

---

## 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `id_` | `uint32_t` | 流标识符 |
| `owner_` | `weak_ptr<core>` | 所属 core 的弱引用 |
| `mr_` | `resource_pointer` | PMR 内存资源 |
| `target_` | `shared_transmission` | 已连接的目标传输层 |
| `closed_` | `bool` | 关闭标志，close() 幂等性保证 |
| `read_size_` | `size_t` | 单次从 target 读取上限 |
| `mux_closed_` | `atomic<bool>` | mux 端已半关闭 |
| `target_closed_` | `atomic<bool>` | target 端已半关闭 |
| `write_channel_` | `concurrent_channel` | 客户端上传方向写通道 |

## 生命周期

```
activate_stream → duct 构造 → start()
  → target_read_loop (target → mux)
  → target_write_loop (mux → target)
  → 任一端 EOF → 半关闭 → 两端均关闭 → close() → 析构
```

## 设计约束

- 通过 shared_from_this 保活，协程持有 self 防止提前析构
- owner_ (weak_ptr) 不与 core 的 ducts_ (shared_ptr) 构成循环引用
- write_channel_ 有界容量提供反压，消除队头阻塞
- target 对象在 close() 后不立即释放，等待 completion handler 执行完毕
