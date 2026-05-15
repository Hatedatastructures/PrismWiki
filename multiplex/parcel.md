---
title: "parcel — 多路复用 UDP 数据报管道"
source: "include/prism/multiplex/parcel.hpp"
implementation: "src/prism/multiplex/parcel.cpp"
module: "multiplex"
type: api
tags: [multiplex, parcel, udp, 数据报, 中继]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/core|core]]"
  - "[[multiplex/duct|duct]]"
  - "[[multiplex/smux/frame|smux::frame]]"
  - "[[resolve/router|router]]"
  - "[[memory/container|container]]"
  - "[[multiplex/smux/craft|smux::craft]]"
  - "[[multiplex/yamux/craft|yamux::craft]]"
---

# parcel.hpp

> 源码: `include/prism/multiplex/parcel.hpp`
> 实现: `src/prism/multiplex/parcel.cpp`
> 模块: [[multiplex/core|multiplex]]

## 概述

多路复用 UDP 数据报管道。协议无关的 UDP 数据报中继，与 [[multiplex/duct|duct]] 的面向连接模型不同，parcel 是无连接的请求-响应模型：每个 PSH 帧承载一个完整 SOCKS5 UDP relay 格式数据报（包含目标地址和负载数据），可发往不同目标地址和端口。

- **on_mux_data** — 解析 SOCKS5 格式数据报，通过 `egress_socket_` 发送到目标
- **downlink_loop** — 独立协程读取 UDP 响应，编码回传 mux 客户端
- **process_buffer** — 从 mux_buffer_ 流式解析完整 UDP 数据报，处理跨 PSH 帧的拆分

空闲超时通过 `idle_timer_` 管理，超时后自动 close()。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[multiplex/core|core]] | 通过 weak_ptr 持有 owner，调用 send_data/remove_parcel |
| 依赖 | [[resolve/router|router]] | DNS 解析目标主机名 |
| 依赖 | [[multiplex/smux/frame|smux::frame]] | 解析/构建 UDP 数据报格式 |
| 依赖 | [[memory/container|container]] | PMR 容器 |
| 被依赖 | [[multiplex/core|core]] | core::parcels_ 持有 shared_ptr<parcel> |
| 被依赖 | [[multiplex/smux/craft|smux::craft]] | activate_stream 创建 parcel |
| 被依赖 | [[multiplex/yamux/craft|yamux::craft]] | activate_stream 创建 parcel |

## 命名空间

`psm::multiplex`

---

## 类: parcel

> 源码: `include/prism/multiplex/parcel.hpp:59`
> 实现: `src/prism/multiplex/parcel.cpp:16`

### 概述

管理单条 UDP 流的完整生命周期，从 activate_stream 解析出 UDP 目标地址后创建，到空闲超时或 mux 会话结束时关闭。每个 PSH 帧的载荷为 SOCKS5 UDP relay 格式数据报：`[ATYP 1B][Addr(var)][Port 2B][Data]`，parcel 解析后通过 egress_socket_ 发送到目标，响应数据编码为相同格式通过 owner_->send_data 发回 mux 客户端。

### 类层次

```
std::enable_shared_from_this<parcel>
  └── parcel
```

---

### parcel::parcel (构造函数)

**功能说明**

构造 parcel 实例。初始化流标识符、owner 弱引用、路由器、executor 缓存和 PMR 内存资源。egress_socket_ 延迟创建，首次发送数据报时按目标协议族（IPv4/IPv6）初始化。packet_addr=true 时每帧使用 SOCKS5 地址格式，packet_addr=false 时使用 length-prefixed 格式。

**签名**

```cpp
parcel(std::uint32_t stream_id, std::shared_ptr<core> owner,
       resolve::router &router,
       std::uint32_t udp_idle_timeout, std::uint32_t udp_max_dg,
       memory::resource_pointer mr, bool packet_addr = false);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符，由 mux 协议在 SYN 帧中分配 |
| `owner` | `shared_ptr<core>` | 所属 core 的共享指针 |
| `router` | `router&` | 路由器引用，用于 DNS 解析 |
| `udp_idle_timeout` | `uint32_t` | UDP 管道空闲超时（毫秒） |
| `udp_max_dg` | `uint32_t` | UDP 数据报最大长度（字节） |
| `mr` | `resource_pointer` | PMR 内存资源 |
| `packet_addr` | `bool` | 是否为 PacketAddr 模式 |

**返回值**

构造函数，无返回值。

**调用（向下）**

- `owner->executor()` — 缓存 executor

**被调用（向上）**

- `make_parcel()` — 工厂函数调用构造函数
- `smux::craft::activate_stream()` — UDP 连接成功后创建
- `yamux::craft::activate_stream()` — UDP 连接成功后创建

**知识域**

延迟 socket 创建、PacketAddr 模式、executor 缓存。

---

### parcel::start

**功能说明**

启动空闲超时监控。通过 co_spawn 启动 uplink_loop 协程，监控 parcel 数据活动。空闲超时由 config::udp_idle_timeout_ms 控制，超时后自动 close()。

**签名**

```cpp
void start();
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `touch_idle_timer()` — 初始化空闲计时器
- `co_spawn(uplink_loop())` — 启动超时监控协程

**被调用（向上）**

- `smux::craft::activate_stream()` — 创建 parcel 后启动
- `yamux::craft::activate_stream()` — 创建 parcel 后启动

**知识域**

co_spawn 协程启动、shared_from_this 保活。

---

### parcel::on_mux_data

**功能说明**

接收 mux 数据报并转发到目标。累积数据到 mux_buffer_，当缓冲区超过最大数据报大小时关闭管道防止内存膨胀。如果没有处理循环在运行，启动 process_buffer 协程。每次调用重置空闲计时器。由 craft::dispatch_push 通过 co_spawn 非阻塞调用。

**签名**

```cpp
auto on_mux_data(std::span<const std::byte> data) -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `span<const byte>` | SOCKS5 UDP relay 格式数据 |

**返回值**

`net::awaitable<void>` — 协程，数据累积后完成。

**调用（向下）**

- `touch_idle_timer()` — 重置空闲计时器
- `co_spawn(process_buffer())` — 启动缓冲区处理循环

**被调用（向上）**

- `smux::craft::dispatch_push()` — 通过 co_spawn 非阻塞调用
- `yamux::craft::dispatch_data()` — 通过 co_spawn 非阻塞调用

**知识域**

缓冲累积模式、processing_ 标志防并发、内存膨胀保护。

---

### parcel::close

**功能说明**

关闭管道（幂等）。首次调用时：标记 closed_ 为 true，关闭 egress_socket_ 和 idle_timer_，调用 owner_->remove_parcel 从 core 的 parcels_ 映射中移除自身。多次调用无副作用。

**签名**

```cpp
void close();
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `egress_socket_->cancel()` — 取消挂起的异步操作
- `egress_socket_->close()` — 关闭 UDP socket
- `idle_timer_.cancel()` — 取消空闲计时器
- `owner_->remove_parcel()` — 从 core 映射移除自身

**被调用（向上）**

- `parcel::~parcel()` — 析构函数中调用
- `parcel::uplink_loop()` — 空闲超时时调用
- `parcel::on_mux_data()` — 缓冲区过大时调用
- `core::close()` — 关闭所有 parcel
- `smux::craft::handle_fin()` — 收到 FIN 帧时调用
- `yamux::craft::handle_fin()` — 收到 FIN 标志时调用
- `yamux::craft::handle_rst()` — 收到 RST 标志时调用

**知识域**

幂等关闭模式、socket 生命周期、egress_socket_ 延迟释放。

---

### parcel::stream_id

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

### parcel::set_destination

**功能说明**

设置无 PacketAddr 模式的固定目标地址。仅在 packet_addr=false 模式下有效，目标地址在 SYN 阶段解析，后续所有数据报发送到同一目标。

**签名**

```cpp
void set_destination(std::string_view host, std::uint16_t port);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `host` | `string_view` | 目标主机名 |
| `port` | `uint16_t` | 目标端口 |

**返回值**

`void`

**调用（向下）**

无。

**被调用（向上）**

- `smux::craft::activate_stream()` — 非 PacketAddr 模式时设置
- `yamux::craft::activate_stream()` — 非 PacketAddr 模式时设置

**知识域**

PacketAddr 模式与 length-prefixed 模式的区别。

---

### parcel::uplink_loop (私有)

**功能说明**

空闲超时监控循环。启动 idle_timer_ 定时器，每次 touch_idle_timer 重置超时。超时后调用 close() 关闭管道。通过 shared_from_this 持有 self 保活。

**签名**

```cpp
auto uplink_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，超时或取消后退出。

**调用（向下）**

- `idle_timer_.async_wait()` — 等待超时
- `parcel::close()` — 超时后关闭

**被调用（向上）**

- `parcel::start()` — 通过 co_spawn 启动

**知识域**

定时器协程、operation_aborted 处理。

---

### parcel::process_buffer (私有)

**功能说明**

缓冲区处理循环。从 mux_buffer_ 中流式解析完整 UDP 数据报并逐个中继。sing-mux 客户端可能将一个 UDP 数据报拆成多个 PSH 帧发送（SOCKS5 地址、Length、Payload 各一帧），因此需要缓冲累积后再解析完整数据报。processing_ 标志保证同一时刻只有一个处理循环运行。

**签名**

```cpp
auto process_buffer() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，处理完所有可解析数据后退出。

**调用（向下）**

- `smux::parse_udp_datagram()` — PacketAddr 模式解析
- `smux::parse_udp_length_prefixed()` — length-prefixed 模式解析
- `parcel::do_send()` — 发送单个数据报

**被调用（向上）**

- `parcel::on_mux_data()` — 通过 co_spawn 启动

**知识域**

缓冲区 swap 优化、快路径/慢路径、数据报跨帧拆分处理。

---

### parcel::do_send (私有)

**功能说明**

发送单个已解析的 UDP 数据报（不等待响应）。通过路由器解析目标端点，确保 UDP socket 可用，然后 async_send_to 发送。首次发送时启动 downlink_loop 协程异步读取响应并回传。

**签名**

```cpp
auto do_send(const memory::string &target_host, std::uint16_t target_port,
             std::span<const std::byte> payload) -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `target_host` | `const memory::string&` | 目标主机 |
| `target_port` | `uint16_t` | 目标端口 |
| `payload` | `span<const byte>` | UDP 负载数据 |

**返回值**

`net::awaitable<void>` — 协程，发送完成后返回。

**调用（向下）**

- `router_.resolve_datagram_target()` — DNS 解析目标端点
- `parcel::ensure_socket()` — 确保 UDP socket 可用
- `egress_socket_->async_send_to()` — 发送 UDP 数据报
- `co_spawn(downlink_loop())` — 首次发送时启动接收循环

**被调用（向上）**

- `parcel::process_buffer()` — 逐个数据报调用

**知识域**

DNS 解析、UDP socket 管理、不可路由地址过滤。

---

### parcel::downlink_loop (私有)

**功能说明**

下行中继循环。独立协程，持续从 egress_socket_ 读取 UDP 响应，编码为 SOCKS5/length-prefixed 格式后通过 owner_->send_data 回传。由 do_send 首次发送时启动，socket 重建或关闭时自动退出。

**签名**

```cpp
auto downlink_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，socket 关闭或错误时退出。

**调用（向下）**

- `egress_socket_->async_receive_from()` — 读取 UDP 响应
- `smux::build_udp_datagram()` — PacketAddr 模式编码
- `smux::build_udp_length_prefixed()` — length-prefixed 模式编码
- `owner_->send_data()` — 通过 mux 帧回传响应

**被调用（向上）**

- `parcel::do_send()` — 首次发送时通过 co_spawn 启动

**知识域**

UDP 响应接收、编码格式选择、weak_ptr lock 检查。

---

### parcel::touch_idle_timer (私有)

**功能说明**

重置空闲计时器。将 idle_timer_ 的超时时间重新设置为 udp_idle_timeout_ms_，延迟管道的空闲关闭。

**签名**

```cpp
void touch_idle_timer();
```

**参数**

无。

**返回值**

`void`

**调用（向下）**

- `idle_timer_.expires_after()` — 设置超时时间

**被调用（向上）**

- `parcel::start()` — 初始化时调用
- `parcel::on_mux_data()` — 每次数据活动时调用

**知识域**

定时器重置模式。

---

### parcel::ensure_socket (私有)

**功能说明**

确保 UDP socket 可用。如果 egress_socket_ 已存在且协议匹配则直接返回 true；否则关闭旧 socket 并创建新的 UDP socket，绑定到任意端口。协议切换时（目标从 IPv4 变为 IPv6 或反之）自动重建 socket。

**签名**

```cpp
auto ensure_socket(net::ip::udp::endpoint::protocol_type protocol) -> net::awaitable<bool>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `protocol` | `udp::endpoint::protocol_type` | 目标协议类型（IPv4/IPv6） |

**返回值**

`net::awaitable<bool>` — true 表示 socket 可用，false 表示创建失败。

**调用（向下）**

- `egress_socket_->cancel()` — 取消旧 socket
- `egress_socket_->close()` — 关闭旧 socket
- `egress_socket_.emplace()` — 创建新 socket
- `egress_socket_->bind()` — 绑定到任意端口

**被调用（向上）**

- `parcel::do_send()` — 发送前确保 socket 可用

**知识域**

UDP socket 生命周期、协议族切换、Windows WSAEINVAL 修复。

---

## 自由函数: make_parcel

**功能说明**

创建 parcel 共享指针的工厂函数。

**签名**

```cpp
[[nodiscard]] inline auto make_parcel(std::uint32_t stream_id, std::shared_ptr<core> owner,
                                      resolve::router &router, std::uint32_t udp_idle_timeout, std::uint32_t udp_max_dg,
                                      memory::resource_pointer mr = {}, bool packet_addr = false)
    -> std::shared_ptr<parcel>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |
| `owner` | `shared_ptr<core>` | 所属 core 的共享指针 |
| `router` | `router&` | 路由器引用 |
| `udp_idle_timeout` | `uint32_t` | UDP 管道空闲超时（毫秒） |
| `udp_max_dg` | `uint32_t` | UDP 数据报最大长度（字节） |
| `mr` | `resource_pointer` | PMR 内存资源 |
| `packet_addr` | `bool` | 是否为 PacketAddr 模式 |

**返回值**

`std::shared_ptr<parcel>` — parcel 的共享指针。

**调用（向下）**

- `std::make_shared<parcel>()` — 构造 parcel 实例

**被调用（向上）**

- `smux::craft::activate_stream()` — UDP 连接成功后创建
- `yamux::craft::activate_stream()` — UDP 连接成功后创建

**知识域**

工厂函数模式。

---

## 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `id_` | `uint32_t` | 流标识符 |
| `owner_` | `weak_ptr<core>` | 所属 core 的弱引用 |
| `router_` | `router&` | 路由器引用 |
| `executor_` | `any_io_executor` | 缓存的 executor |
| `udp_idle_timeout_ms_` | `uint32_t` | UDP 管道空闲超时 |
| `udp_max_datagram_` | `uint32_t` | UDP 数据报最大长度 |
| `mr_` | `resource_pointer` | PMR 内存资源 |
| `closed_` | `bool` | 关闭标志 |
| `packet_addr_` | `bool` | PacketAddr 模式标志 |
| `destination_host_` | `memory::string` | 无 PacketAddr 模式时的目标主机 |
| `destination_port_` | `uint16_t` | 无 PacketAddr 模式时的目标端口 |
| `idle_timer_` | `steady_timer` | 空闲超时计时器 |
| `egress_socket_` | `optional<udp::socket>` | 出站 UDP socket |
| `recv_buffer_` | `memory::vector<byte>` | UDP 响应接收缓冲区 |
| `mux_buffer_` | `memory::vector<byte>` | mux 数据累积缓冲区 |
| `processing_` | `atomic<bool>` | 缓冲区处理标志 |
| `recv_running_` | `atomic<bool>` | recv_loop 运行标志 |

## 设计约束

- owner_ (weak_ptr) 不构成循环引用
- on_mux_data 中串行处理，同一时刻只有一个数据报在处理中
- egress_socket_ 延迟创建，按目标协议族（IPv4/IPv6）初始化
- 缓冲区超过最大数据报大小时自动关闭，防止内存膨胀
