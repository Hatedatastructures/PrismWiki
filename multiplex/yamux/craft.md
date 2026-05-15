---
title: "craft — yamux 多路复用会话服务端"
source: "include/prism/multiplex/yamux/craft.hpp"
implementation: "src/prism/multiplex/yamux/craft.cpp"
module: "multiplex"
type: api
tags: [multiplex, yamux, craft, 会话服务端, 流量控制]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/core|core]]"
  - "[[multiplex/yamux/frame|yamux::frame]]"
  - "[[multiplex/yamux/config|yamux::config]]"
  - "[[multiplex/config|config]]"
  - "[[multiplex/duct|duct]]"
  - "[[multiplex/parcel|parcel]]"
  - "[[multiplex/smux/frame|smux::frame]]"
  - "[[multiplex/bootstrap|bootstrap]]"
  - "[[channel/transport/reliable|reliable]]"
  - "[[resolve/router|router]]"
---

# craft.hpp — yamux 会话服务端

> 源码: `include/prism/multiplex/yamux/craft.hpp`
> 实现: `src/prism/multiplex/yamux/craft.cpp`
> 模块: [[multiplex/config|multiplex]] > yamux

## 概述

yamux 多路复用会话服务端。继承 [[multiplex/core|core]]，实现 yamux 协议服务端逻辑。兼容 Hashicorp/yamux + sing-mux 协商。

与 smux 相比，yamux 提供：
- 完整的流量控制（256KB 初始窗口）
- 标志位系统（SYN/ACK/FIN/RST）
- Ping 心跳机制
- 12 字节大端帧头（vs smux 8 字节小端）

### 流打开流程

1. 客户端发送 WindowUpdate(SYN)
2. 服务端创建 pending，回复 WindowUpdate(ACK)
3. 客户端发送 Data(none) 携带目标地址
4. 服务端解析地址连接目标，发送 Data(none) 携带 0x00 成功状态
5. 创建 [[multiplex/duct|duct]]/[[multiplex/parcel|parcel]]

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[multiplex/core|core]] | 多路复用核心基类 |
| 依赖 | [[multiplex/yamux/frame|frame]] | yamux 帧格式定义与编解码 |
| 依赖 | [[multiplex/yamux/config|yamux::config]] | yamux 配置参数 |
| 依赖 | [[multiplex/config|config]] | 多路复用总配置 |
| 依赖 | [[multiplex/smux/frame|smux::frame]] | 复用 parse_mux_address 解析地址 |
| 依赖 | [[multiplex/duct|duct]] | TCP 流管道 |
| 依赖 | [[multiplex/parcel|parcel]] | UDP 数据报管道 |
| 依赖 | [[channel/transport/reliable|reliable]] | 创建可靠传输层 |
| 依赖 | [[resolve/router|router]] | 路由器，连接目标 |
| 被依赖 | [[multiplex/bootstrap|bootstrap]] | 协商后创建 yamux::craft 实例 |

## 命名空间

`psm::multiplex::yamux`

---

## 结构体: outbound_frame

**功能说明**

出站帧结构，header 与 payload 分离传递。header 为 12 字节编码后的帧头，payload 持有实际数据。两者分离传递至 send_loop，由 send_loop 分别写入 transport，消除 memcpy 开销。

| 字段 | 类型 | 说明 |
|------|------|------|
| `header` | `array<byte, 12>` | 编码后的帧头 |
| `payload` | `vector<byte>` | 帧载荷（所有权转移） |

---

## 结构体: stream_window

**功能说明**

流窗口状态，用于流量控制。跟踪单条流的发送和接收窗口，使用原子变量确保线程安全。window_signal 用于在窗口不足时等待 WindowUpdate 帧唤醒发送方。

| 字段 | 类型 | 说明 |
|------|------|------|
| `send_window` | `atomic<uint32_t>` | 发送窗口（初始 256KB） |
| `recv_consumed` | `atomic<uint32_t>` | 已消费的接收数据量 |
| `window_signal` | `shared_ptr<steady_timer>` | 窗口更新信号 |

---

## 类: craft

> 源码: `include/prism/multiplex/yamux/craft.hpp:78`
> 实现: `src/prism/multiplex/yamux/craft.cpp:20`

### 概述

yamux 多路复用会话服务端。继承 core，实现 yamux 协议服务端逻辑，包括帧读写、窗口管理、心跳和流生命周期管理。

### 类层次

```
core (抽象基类)
  └── craft (final)
```

---

### craft::craft (构造函数)

**功能说明**

构造 yamux 会话。初始化传输层、配置和发送通道，会话处于未启动状态，调用 start() 后才会进入协议主循环。

**签名**

```cpp
craft(channel::transport::shared_transmission transport, resolve::router &router,
      const multiplex::config &cfg, memory::resource_pointer mr = {});
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `shared_transmission` | 已建立的传输层连接 |
| `router` | `router&` | 路由器引用 |
| `cfg` | `const multiplex::config&` | 多路复用配置参数 |
| `mr` | `resource_pointer` | PMR 内存资源 |

**返回值**

构造函数，无返回值。

**调用（向下）**

- `core::core()` — 调用基类构造函数
- `channel_` 初始化 — 有界发送通道，容量为 max_streams

**被调用（向上）**

- `std::make_shared<yamux::craft>()` — 由 bootstrap 创建

**知识域**

基类构造、有界通道初始化。

---

### craft::send_data

**功能说明**

发送 Data 帧到客户端。扣减发送窗口，窗口不足时等待 WindowUpdate 帧唤醒。将 payload 编码为 outbound_frame 推入 channel_ 发送通道。

**签名**

```cpp
auto send_data(std::uint32_t stream_id, memory::vector<std::byte> payload) const
    -> net::awaitable<void> override;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 目标流标识符 |
| `payload` | `memory::vector<byte>` | 要发送的数据（所有权转移） |

**返回值**

`net::awaitable<void>` — 协程，窗口获取成功且帧推入通道后完成。

**调用（向下）**

- `get_window()` — 获取流窗口
- `window_signal->async_wait()` — 窗口不足时等待唤醒
- `craft::push_frame()` — 推入发送通道

**被调用（向上）**

- `core::send_data()` 虚函数接口 — 由 duct/parcel 调用
- `craft::activate_stream()` — 发送成功/错误状态
- `craft::dispatch_data()` — pending 流中更新接收窗口

**知识域**

流量控制、窗口扣减、CAS 操作、等待-唤醒模式。

---

### craft::send_fin

**功能说明**

发送 FIN 帧到客户端。通过 co_spawn 异步发送，不阻塞调用者。

**签名**

```cpp
void send_fin(std::uint32_t stream_id) override;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 目标流标识符 |

**返回值**

`void` — 异步发送，立即返回。

**调用（向下）**

- `craft::push_frame()` — 通过 co_spawn 异步调用

**被调用（向上）**

- `core::send_fin()` 虚函数接口 — 由 duct::target_read_loop 调用
- `craft::activate_stream()` — 连接失败时调用

**知识域**

co_spawn 异步发送。

---

### craft::executor

**功能说明**

获取 transport executor。

**签名**

```cpp
[[nodiscard]] net::any_io_executor executor() const override;
```

**参数**

无。

**返回值**

`net::any_io_executor` — transport 的执行器。

**调用（向下）**

- `transport_->executor()` — 返回底层传输的执行器

**被调用（向上）**

- `core::executor()` 虚函数接口

**知识域**

Boost.Asio executor 模型。

---

### craft::remove_duct

**功能说明**

移除 TCP 管道并清理窗口状态。override core::remove_duct，额外清理 windows_ 映射，唤醒可能在 send_data 中等待窗口的协程。

**签名**

```cpp
void remove_duct(std::uint32_t stream_id) override;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 要移除的流标识符 |

**返回值**

`void`

**调用（向下）**

- `window_signal->cancel()` — 唤醒等待窗口的协程
- `windows_.erase()` — 清理窗口映射
- `core::remove_duct()` — 调用基类移除

**被调用（向上）**

- `duct::close()` — 通过 owner_->remove_duct 调用

**知识域**

窗口清理、协程唤醒。

---

### craft::remove_parcel

**功能说明**

移除 UDP 管道并清理窗口状态。override core::remove_parcel，额外清理 windows_ 映射。

**签名**

```cpp
void remove_parcel(std::uint32_t stream_id) override;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 要移除的流标识符 |

**返回值**

`void`

**调用（向下）**

- `window_signal->cancel()` — 唤醒等待窗口的协程
- `windows_.erase()` — 清理窗口映射
- `core::remove_parcel()` — 调用基类移除

**被调用（向上）**

- `parcel::close()` — 通过 owner_->remove_parcel 调用

**知识域**

窗口清理、协程唤醒。

---

### craft::close

**功能说明**

关闭 yamux 会话（幂等）。先取消 pending 超时定时器防止超时协程在 close 过程中运行，然后调用 core::close() 清理 pending/ducts/parcels，最后清理剩余的 windows_。

**签名**

```cpp
void close() override;
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `timer->cancel()` — 取消所有 pending 超时定时器
- `core::close()` — 基类关闭逻辑
- `window_signal->cancel()` — 唤醒所有等待窗口的协程
- `windows_.clear()` — 清理剩余窗口

**被调用（向上）**

- `craft::handle_go_away()` — 收到 GoAway 时调用
- `craft::send_loop()` — 写入失败时调用
- `craft::frame_loop()` 退出后 — 通过 run() 的异常处理

**知识域**

定时器清理顺序、幂等关闭。

---

### craft::run (私有)

**功能说明**

协议主循环。启动 send_loop 和 ping_loop，然后进入 frame_loop。

**签名**

```cpp
auto run() -> net::awaitable<void> override;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `co_spawn(send_loop())` — 启动发送循环
- `co_spawn(ping_loop())` — 启动心跳循环（当 enable_ping 且 ping_interval_ms > 0）
- `craft::frame_loop()` — 进入帧循环
- `channel_.cancel()` — 帧循环退出后取消发送通道

**被调用（向上）**

- `core::start()` — 通过 co_spawn 启动

**知识域**

协程生命周期管理。

---

### craft::frame_loop (私有)

**功能说明**

帧循环主协程。循环读取帧头（12 字节）+ 载荷（Data 帧才有），按消息类型分发：Data 由 handle_data 根据 flags 分发，WindowUpdate 由 handle_window_update 处理流创建或窗口更新，Ping 由 handle_ping 回复 ACK，GoAway 由 handle_go_away 关闭会话。

**签名**

```cpp
auto frame_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `transport_->async_read()` — 读取帧头和载荷
- `yamux::parse_header()` — 解析帧头
- `craft::handle_data()` — 处理 Data 帧
- `craft::handle_window_update()` — 处理 WindowUpdate 帧
- `craft::handle_ping()` — 处理 Ping 帧
- `craft::handle_go_away()` — 处理 GoAway 帧
- `craft::push_frame()` — 发送 GoAway（大帧时）

**被调用（向上）**

- `craft::run()` — 主循环中调用

**知识域**

帧循环模式、消息类型分发、大帧 OOM 保护。

---

### craft::handle_data (私有)

**功能说明**

处理 Data 帧。根据 flags 分发：SYN → handle_syn，RST → handle_rst，FIN → handle_fin，无标志 → dispatch_data。

**签名**

```cpp
auto handle_data(const frame_header &hdr, memory::vector<std::byte> payload)
    -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `hdr` | `const frame_header&` | 帧头 |
| `payload` | `memory::vector<byte>` | 帧载荷（所有权转移） |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `craft::handle_syn()` — SYN 标志
- `craft::handle_rst()` — RST 标志
- `craft::handle_fin()` — FIN 标志
- `craft::dispatch_data()` — 无标志（纯数据）

**被调用（向上）**

- `craft::frame_loop()` — 收到 Data 帧时调用

**知识域**

yamux 标志位语义、Data 帧复用。

---

### craft::handle_syn (私有)

**功能说明**

处理 Data(SYN) 帧（sing-mux 兼容模式）。检查 max_streams 限制，创建 pending_entry，回复 WindowUpdate(ACK)，payload 非空时累积到 buffer 并尝试激活。

**签名**

```cpp
auto handle_syn(std::uint32_t stream_id, memory::vector<std::byte> payload)
    -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |
| `payload` | `memory::vector<byte>` | 帧载荷（可能携带地址数据） |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `pending_.emplace()` — 创建 pending_entry
- `get_or_create_window()` — 创建流窗口
- `craft::start_pending_timeout()` — 设置 pending 超时
- `craft::push_frame()` — 发送 WindowUpdate(ACK)
- `craft::try_activate_pending()` — 尝试激活

**被调用（向上）**

- `craft::handle_data()` — 收到 Data(SYN) 时调用

**知识域**

sing-mux 兼容模式、pending 状态管理。

---

### craft::handle_rst (私有)

**功能说明**

处理 RST 标志（强制重置流）。清除 pending，调用 duct/parcel 的关闭方法，唤醒等待窗口的协程，清理窗口状态。

**签名**

```cpp
void handle_rst(std::uint32_t stream_id);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 要重置的流标识符 |

**返回值**

`void`

**调用（向下）**

- `pending_.erase()` — 清除 pending 流
- `duct::on_mux_fin()` — 半关闭 TCP 流
- `parcel::close()` — 关闭 UDP 流
- `window_signal->cancel()` — 唤醒等待窗口的协程
- `windows_.erase()` — 清理窗口

**被调用（向上）**

- `craft::handle_data()` — 收到 Data(RST) 时调用

**知识域**

RST 强制重置语义。

---

### craft::handle_fin (私有)

**功能说明**

处理 FIN 标志（半关闭流）。按顺序检查：pending（直接移除）→ duct（on_mux_fin）→ parcel（close）。

**签名**

```cpp
void handle_fin(std::uint32_t stream_id);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 要半关闭的流标识符 |

**返回值**

`void`

**调用（向下）**

- `pending_.erase()` — 移除 pending 流
- `duct::on_mux_fin()` — 半关闭 TCP 流
- `parcel::close()` — 关闭 UDP 流
- `windows_.erase()` — 清理窗口

**被调用（向上）**

- `craft::handle_data()` — 收到 Data(FIN) 时调用

**知识域**

半关闭语义。

---

### craft::dispatch_data (私有)

**功能说明**

分发纯数据帧到 pending/duct/parcel。三路分发逻辑：
1. pending 流：累积数据，尝试激活，更新接收窗口
2. 已连接 duct：co_spawn(detached) 调用 on_mux_data
3. 活跃 parcel：co_spawn(detached) 调用 on_mux_data
4. 流不存在时发送 WindowUpdate(RST)

**签名**

```cpp
auto dispatch_data(std::uint32_t stream_id, memory::vector<std::byte> payload)
    -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |
| `payload` | `memory::vector<byte>` | 帧载荷（所有权转移） |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `craft::try_activate_pending()` — 尝试激活 pending 流
- `craft::update_recv_window()` — 更新接收窗口
- `duct::on_mux_data()` — 通过 co_spawn 非阻塞调用
- `parcel::on_mux_data()` — 通过 co_spawn 非阻塞调用
- `craft::push_frame()` — 发送 RST（流不存在时）

**被调用（向上）**

- `craft::handle_data()` — 无标志 Data 帧时调用

**知识域**

三路分发模式、接收窗口更新。

---

### craft::try_activate_pending (私有)

**功能说明**

尝试激活 pending 流。当 buffer >= 7 字节且未连接时，通过 co_spawn 启动 activate_stream。connecting 标志防止重复激活。

**签名**

```cpp
void try_activate_pending(std::uint32_t stream_id);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`void`

**调用（向下）**

- `co_spawn(activate_stream())` — 启动连接

**被调用（向上）**

- `craft::handle_syn()` — SYN 后尝试激活
- `craft::dispatch_data()` — 数据累积后尝试激活

**知识域**

connecting 防重复标志、最小地址长度检查。

---

### craft::handle_window_update (私有)

**功能说明**

处理 WindowUpdate 帧。处理多种情况：stream_id==0 时为会话级窗口更新（忽略），RST 标志重置流，FIN 标志半关闭流，SYN（无 ACK）为客户端打开新流（回复 ACK），SYN+ACK 为确认流创建，普通帧增加 send_window 并唤醒等待方。

**签名**

```cpp
auto handle_window_update(const frame_header &hdr) -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `hdr` | `const frame_header&` | 帧头（length 字段为窗口增量） |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `pending_.emplace()` — SYN 时创建 pending 流
- `get_or_create_window()` — 创建/获取流窗口
- `craft::start_pending_timeout()` — 设置 pending 超时
- `craft::push_frame()` — 发送 ACK/RST
- `duct::on_mux_fin()` — FIN 时半关闭
- `parcel::close()` — RST 时关闭
- `window_signal->cancel()` — 唤醒等待窗口的协程

**被调用（向上）**

- `craft::frame_loop()` — 收到 WindowUpdate 帧时调用

**知识域**

yamux 流打开流程、窗口管理、标志位组合语义。

---

### craft::handle_ping (私有)

**功能说明**

处理 Ping 帧。SYN 标志为请求，回复 Ping(ACK) 并携带相同 ID；ACK 标志为响应，忽略。

**签名**

```cpp
auto handle_ping(const frame_header &hdr) const -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `hdr` | `const frame_header&` | 帧头（length 字段为 ping ID） |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `craft::push_frame()` — 发送 Ping(ACK)

**被调用（向上）**

- `craft::frame_loop()` — 收到 Ping 帧时调用

**知识域**

Ping 心跳请求/响应。

---

### craft::handle_go_away (私有)

**功能说明**

处理 GoAway 帧。收到 GoAway 后调用 close() 关闭整个会话。

**签名**

```cpp
auto handle_go_away(const frame_header &hdr) -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `hdr` | `const frame_header&` | 帧头（length 字段为终止原因码） |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `core::close()` — 关闭会话

**被调用（向上）**

- `craft::frame_loop()` — 收到 GoAway 帧时调用

**知识域**

GoAway 会话终止语义。

---

### craft::activate_stream (私有)

**功能说明**

从 pending 解析地址、连接目标、创建 duct/parcel。解析 SOCKS5 格式目标地址（复用 smux::parse_mux_address），区分 TCP/UDP。

**签名**

```cpp
auto activate_stream(std::uint32_t stream_id) -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `smux::parse_mux_address()` — 解析目标地址
- `router_.async_forward()` — 连接目标
- `channel::transport::make_reliable()` — 创建可靠传输层
- `make_duct()` — 创建 TCP 管道
- `make_parcel()` — 创建 UDP 管道
- `craft::send_data()` — 发送成功/错误状态
- `craft::send_fin()` — 连接失败时发送 FIN

**被调用（向上）**

- `craft::try_activate_pending()` — 通过 co_spawn 启动

**知识域**

地址解析、TCP/UDP 分流、连接失败处理。

---

### craft::start_pending_timeout (私有)

**功能说明**

为 pending 流设置超时定时器。当 stream_open_timeout_ms > 0 时创建定时器，超时后检查流是否仍在 pending 中，若是则清理并发送 RST。

**签名**

```cpp
void start_pending_timeout(std::uint32_t stream_id);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`void`

**调用（向下）**

- `steady_timer::async_wait()` — 等待超时
- `craft::push_frame()` — 超时后发送 RST

**被调用（向上）**

- `craft::handle_syn()` — SYN 后设置超时
- `craft::handle_window_update()` — SYN via WindowUpdate 后设置超时

**知识域**

pending 超时机制、定时器管理。

---

### craft::get_or_create_window (私有)

**功能说明**

获取或创建流窗口。若 windows_ 中不存在指定流的窗口则创建一个新的。

**签名**

```cpp
stream_window *get_or_create_window(std::uint32_t stream_id);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`stream_window*` — 流窗口指针，永不返回 nullptr。

**调用（向下）**

- `windows_.emplace()` — 创建新窗口

**被调用（向上）**

- `craft::handle_syn()` — SYN 后创建窗口
- `craft::handle_window_update()` — SYN via WindowUpdate 后创建窗口
- `craft::update_recv_window()` — 更新接收窗口时获取

**知识域**

窗口映射管理。

---

### craft::get_window (私有)

**功能说明**

获取流窗口（不创建）。在 windows_ 映射中查找指定流的窗口，不创建新条目。

**签名**

```cpp
[[nodiscard]] stream_window *get_window(std::uint32_t stream_id) const;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`stream_window*` — 流窗口指针，不存在时返回 nullptr。

**调用（向下）**

无。

**被调用（向上）**

- `craft::send_data()` — 检查发送窗口
- `craft::handle_window_update()` — 普通窗口更新时查找

**知识域**

只读查找、nullptr 检查。

---

### craft::update_recv_window (私有)

**功能说明**

检查并更新接收窗口，必要时发送 WindowUpdate。累积 recv_consumed，当达到 initial_stream_window/2 阈值时发送 WindowUpdate(none) 帧。

**签名**

```cpp
auto update_recv_window(std::uint32_t stream_id, std::uint32_t consumed)
    -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |
| `consumed` | `uint32_t` | 本次消费的数据量 |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `get_or_create_window()` — 获取流窗口
- `craft::push_frame()` — 发送 WindowUpdate

**被调用（向上）**

- `craft::dispatch_data()` — 数据分发时更新

**知识域**

接收窗口阈值、累积消费量。

---

### craft::push_frame (私有)

**功能说明**

将帧推送到发送通道。编码帧头为 12 字节数组，与 payload 组装为 outbound_frame，通过 channel_ 发送通道传递给 send_loop。

**签名**

```cpp
auto push_frame(message_type type, flags f, std::uint32_t stream_id,
                std::uint32_t length, memory::vector<std::byte> payload) const
    -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `type` | `message_type` | 消息类型 |
| `f` | `flags` | 标志位 |
| `stream_id` | `uint32_t` | 流标识符 |
| `length` | `uint32_t` | Length 字段值 |
| `payload` | `memory::vector<byte>` | 帧载荷（所有权转移） |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `yamux::build_header()` — 编码帧头
- `channel_.async_send()` — 推入发送通道

**被调用（向上）**

- `craft::send_data()` — 发送 Data 帧
- `craft::send_fin()` — 发送 FIN 帧
- `craft::handle_syn()` — 发送 ACK
- `craft::handle_window_update()` — 发送 ACK/RST
- `craft::handle_ping()` — 发送 Ping(ACK)
- `craft::dispatch_data()` — 发送 RST
- `craft::start_pending_timeout()` — 发送 RST
- `craft::ping_loop()` — 发送 Ping(SYN)

**知识域**

帧编码、通道发送。

---

### craft::send_loop (私有)

**功能说明**

发送循环协程，scatter-gather 写入 transport。从 channel_ 取出 outbound_frame，先写 12 字节帧头，再写 payload。

**签名**

```cpp
auto send_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `channel_.async_receive()` — 从通道取帧
- `transport_->async_write_scatter()` — scatter-gather 写入
- `transport_->async_write()` — 纯帧头写入
- `core::close()` — 写入失败时关闭

**被调用（向上）**

- `craft::run()` — 通过 co_spawn 启动

**知识域**

scatter-gather I/O。

---

### craft::ping_loop (私有)

**功能说明**

主动 Ping 心跳循环。当 enable_ping 为 true 且 ping_interval_ms > 0 时运行，按配置间隔发送 Ping(SYN) 帧检测连接活性。

**签名**

```cpp
auto ping_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `steady_timer::async_wait()` — 等待心跳间隔
- `craft::push_frame()` — 发送 Ping(SYN)

**被调用（向上）**

- `craft::run()` — 通过 co_spawn 启动

**知识域**

主动心跳机制、ping_id 原子递增。

---

## 与 smux 的差异

| 特性 | smux | yamux |
|------|------|-------|
| 帧头大小 | 8 字节 | 12 字节 |
| 字节序 | 小端 | 大端 |
| 流量控制 | 无 | 256KB 窗口 |
| 心跳 | NOP（不回复） | Ping（请求/响应） |
| 标志位 | 无 | SYN/ACK/FIN/RST |
| 流打开 | SYN → PSH(地址) | WindowUpdate(SYN) → Data(地址) |
