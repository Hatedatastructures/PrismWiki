---
title: "craft — smux 多路复用会话服务端"
source: "include/prism/multiplex/smux/craft.hpp"
implementation: "src/prism/multiplex/smux/craft.cpp"
module: "multiplex"
type: api
tags: [multiplex, smux, craft, 会话服务端, sing-mux]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/core|core]]"
  - "[[multiplex/smux/frame|smux::frame]]"
  - "[[multiplex/smux/config|smux::config]]"
  - "[[multiplex/config|config]]"
  - "[[multiplex/duct|duct]]"
  - "[[multiplex/parcel|parcel]]"
  - "[[multiplex/bootstrap|bootstrap]]"
  - "[[channel/transport/reliable|reliable]]"
  - "[[resolve/router|router]]"
---

# craft.hpp — smux 会话服务端

> 源码: `include/prism/multiplex/smux/craft.hpp`
> 实现: `src/prism/multiplex/smux/craft.cpp`
> 模块: [[multiplex/config|multiplex]] > smux

## 概述

smux 多路复用会话服务端。继承 [[multiplex/core|core]]，实现 smux v1 帧协议和 sing-mux 协议协商。兼容 Mihomo/xtaci/smux v1。

帧格式为 8 字节定长帧头 `[Version 1B][Cmd 1B][Length 2B LE][StreamID 4B LE]`，最大帧载荷 65535 字节。

### 协议会话生命周期

1. `run()` 启动 send_loop 和 keepalive_loop
2. `frame_loop()` — 循环读取帧按命令分发
3. SYN → 创建 pending_entry，累积首个 PSH 的地址数据
4. 地址完整 → `activate_stream()` 连接目标创建 [[multiplex/duct|duct]]/[[multiplex/parcel|parcel]]
5. 后续 PSH → `dispatch_push()` 非阻塞分发

### 发送路径（客户端下载方向）

```
duct::target_read_loop() → send_data() → push_frame() → channel_ → send_loop() → transport
```

header 与 payload 分离传递，消除 serialize 的 payload memcpy。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[multiplex/core|core]] | 多路复用核心基类 |
| 依赖 | [[multiplex/smux/frame|frame]] | smux 帧格式定义与编解码 |
| 依赖 | [[multiplex/smux/config|smux::config]] | smux 配置参数 |
| 依赖 | [[multiplex/config|config]] | 多路复用总配置 |
| 依赖 | [[multiplex/duct|duct]] | TCP 流管道 |
| 依赖 | [[multiplex/parcel|parcel]] | UDP 数据报管道 |
| 依赖 | [[channel/transport/reliable|reliable]] | 创建可靠传输层 |
| 依赖 | [[resolve/router|router]] | 路由器，连接目标 |
| 被依赖 | [[multiplex/bootstrap|bootstrap]] | 协商后创建 smux::craft 实例 |

## 命名空间

`psm::multiplex::smux`

---

## 自由函数

### make_data_frame

**功能说明**

构建 DATA (PSH) 帧字节序列。帧格式：`[Version 1B][Cmd=PSH 1B][Length 2B LE][StreamID 4B LE][Payload]`。

**签名**

```cpp
[[nodiscard]] auto make_data_frame(std::uint32_t stream_id,
                                   std::span<const std::byte> payload)
    -> memory::vector<std::byte>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |
| `payload` | `span<const byte>` | 数据负载 |

**返回值**

`memory::vector<std::byte>` — 完整的帧字节序列（8 字节帧头 + payload）。

**调用（向下）**

- 内部 `build_header()` — 构建帧头字节数组

**被调用（向上）**

- 测试代码 — 构建测试帧

**知识域**

smux 帧格式、小端序编码。

---

### make_syn_frame

**功能说明**

构建 SYN 帧字节序列。帧格式：`[Version 1B][Cmd=SYN 1B][Length=0 2B LE][StreamID 4B LE]`。

**签名**

```cpp
[[nodiscard]] auto make_syn_frame(std::uint32_t stream_id)
    -> std::array<std::byte, frame_header_size>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`std::array<std::byte, 8>` — 8 字节 SYN 帧头（无 payload）。

**调用（向下）**

- 内部 `build_header()` — 构建帧头字节数组

**被调用（向上）**

- 测试代码 — 构建测试帧

**知识域**

smux SYN 命令。

---

### make_fin_frame

**功能说明**

构建 FIN 帧字节序列。帧格式：`[Version 1B][Cmd=FIN 1B][Length=0 2B LE][StreamID 4B LE]`。

**签名**

```cpp
[[nodiscard]] auto make_fin_frame(std::uint32_t stream_id)
    -> std::array<std::byte, frame_header_size>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`std::array<std::byte, 8>` — 8 字节 FIN 帧头（无 payload）。

**调用（向下）**

- 内部 `build_header()` — 构建帧头字节数组

**被调用（向上）**

- 测试代码 — 构建测试帧

**知识域**

smux FIN 命令。

---

## 结构体: outbound_frame

**功能说明**

出站帧结构，header 与 payload 分离传递。header 为 8 字节编码后的帧头，payload 持有实际数据。两者分离传递至 send_loop，由 send_loop 分别写入 transport，消除将 header+payload 拼接到单一 buffer 的 memcpy 开销。

| 字段 | 类型 | 说明 |
|------|------|------|
| `header` | `array<byte, 8>` | 编码后的帧头 |
| `payload` | `vector<byte>` | 帧载荷（所有权转移） |

---

## 类: craft

> 源码: `include/prism/multiplex/smux/craft.hpp:84`
> 实现: `src/prism/multiplex/smux/craft.cpp:59`

### 概述

smux 多路复用会话服务端。继承 core，实现 smux v1 帧协议和 sing-mux 协议协商。是 smux 多路复用的协议层实现，负责帧的读写、解析和分发。

### 类层次

```
core (抽象基类)
  └── craft (final)
```

---

### craft::craft (构造函数)

**功能说明**

构造 smux 会话。初始化传输层、配置和发送通道，会话处于未启动状态，调用 start() 后才会进入协议主循环。

**签名**

```cpp
craft(channel::transport::shared_transmission transport, resolve::router &router,
      const multiplex::config &cfg, memory::resource_pointer mr = {});
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `shared_transmission` | 已建立的传输层连接（通常是 Trojan 隧道） |
| `router` | `router&` | 路由器引用，用于解析地址并连接目标 |
| `cfg` | `const multiplex::config&` | 多路复用配置参数 |
| `mr` | `resource_pointer` | PMR 内存资源 |

**返回值**

构造函数，无返回值。

**调用（向下）**

- `core::core()` — 调用基类构造函数
- `channel_` 初始化 — 有界发送通道，容量为 max_streams

**被调用（向上）**

- `std::make_shared<smux::craft>()` — 由 bootstrap 创建

**知识域**

基类构造、有界通道初始化。

---

### craft::send_data

**功能说明**

发送 PSH 帧到客户端。将 payload 编码为 outbound_frame（header 与 payload 分离），推入 channel_ 发送通道。不执行 memcpy，payload 直接 move。

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

`net::awaitable<void>` — 协程，帧推入通道后完成。

**调用（向下）**

- `craft::push_frame()` — 编码帧头并推入 channel_

**被调用（向上）**

- `core::send_data()` 虚函数接口 — 由 duct/parcel 调用
- `craft::activate_stream()` — 发送成功/错误状态

**知识域**

零拷贝传递、所有权转移。

---

### craft::send_fin

**功能说明**

发送 FIN 帧到客户端。通过 co_spawn 异步发送，不阻塞调用者（通常是 duct 的 target_read_loop）。

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

co_spawn 异步发送、不阻塞调用者。

---

### craft::executor

**功能说明**

获取 transport executor，用于 duct/parcel 协程调度。

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

- `core::executor()` 虚函数接口 — 由 duct/parcel 构造函数调用

**知识域**

Boost.Asio executor 模型。

---

### craft::run (私有)

**功能说明**

协议主循环。依次启动 send_loop 和 keepalive_loop，然后进入 frame_loop。frame_loop 退出后取消 channel_。

**签名**

```cpp
auto run() -> net::awaitable<void> override;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，协议会话主循环。

**调用（向下）**

- `co_spawn(send_loop())` — 启动发送循环
- `co_spawn(keepalive_loop())` — 启动心跳循环（当 keepalive_interval_ms > 0）
- `craft::frame_loop()` — 进入帧循环
- `channel_.cancel()` — 帧循环退出后取消发送通道

**被调用（向上）**

- `core::start()` — 通过 co_spawn 启动

**知识域**

协程生命周期管理、detached 协程。

---

### craft::frame_loop (私有)

**功能说明**

帧循环主协程。循环读取帧头（8 字节）+ 载荷，按命令类型分发：SYN 创建 pending_entry，PSH 由 dispatch_push 非阻塞三路分发，FIN 关闭对应流，NOP 忽略。读取失败或无效帧时退出循环。

**签名**

```cpp
auto frame_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，读取失败或会话关闭时退出。

**调用（向下）**

- `transport_->async_read()` — 读取帧头和载荷
- `smux::deserialization()` — 解析帧头
- `craft::handle_syn()` — 处理 SYN 命令
- `craft::dispatch_push()` — 处理 PSH 命令
- `craft::handle_fin()` — 处理 FIN 命令

**被调用（向上）**

- `craft::run()` — 主循环中调用

**知识域**

帧循环模式、命令分发、帧头验证。

---

### craft::handle_syn (私有)

**功能说明**

处理 SYN 帧，创建 pending_entry。检查 max_streams 限制，未超出则在 pending_ 中创建条目，等待后续 PSH 帧累积地址数据。

**签名**

```cpp
auto handle_syn(std::uint32_t stream_id) -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 新建的流标识符 |

**返回值**

`net::awaitable<void>` — 协程。

**调用（向下）**

- `pending_.emplace()` — 创建 pending_entry

**被调用（向上）**

- `craft::frame_loop()` — 收到 SYN 命令时调用

**知识域**

流数量限制、pending 状态管理。

---

### craft::dispatch_push (私有)

**功能说明**

处理 PSH 帧，非阻塞三路分发。三路分发逻辑：
1. pending 流：累积数据到 entry.buffer，数据足够时 spawn activate_stream
2. 已连接 duct：co_spawn(detached) 调用 on_mux_data，不阻塞帧循环
3. 活跃 parcel：co_spawn(detached) 调用 on_mux_data，不阻塞帧循环

**签名**

```cpp
void dispatch_push(std::uint32_t stream_id, memory::vector<std::byte> payload);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |
| `payload` | `memory::vector<byte>` | 帧载荷数据（所有权转移） |

**返回值**

`void`

**调用（向下）**

- `craft::activate_stream()` — 通过 co_spawn 启动连接
- `duct::on_mux_data()` — 通过 co_spawn 非阻塞调用
- `parcel::on_mux_data()` — 通过 co_spawn 非阻塞调用

**被调用（向上）**

- `craft::frame_loop()` — 收到 PSH 命令时调用

**知识域**

三路分发模式、非阻塞 dispatch、detached 协程。

---

### craft::handle_fin (私有)

**功能说明**

处理 FIN 帧。按顺序检查：pending（直接移除）→ duct（on_mux_fin 半关闭）→ parcel（close 完全关闭）。

**签名**

```cpp
void handle_fin(std::uint32_t stream_id);
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 要关闭的流标识符 |

**返回值**

`void`

**调用（向下）**

- `pending_.erase()` — 移除 pending 流
- `duct::on_mux_fin()` — 半关闭 TCP 流
- `parcel::close()` — 关闭 UDP 流

**被调用（向上）**

- `craft::frame_loop()` — 收到 FIN 命令时调用

**知识域**

半关闭语义、流状态查找顺序。

---

### craft::activate_stream (私有)

**功能说明**

从 pending 解析地址、连接目标、创建 duct/parcel。解析 SOCKS5 格式目标地址，区分 TCP/UDP：
- UDP：发送成功状态后创建 parcel 并转发剩余数据
- TCP：通过 router 连接目标后发送状态并创建 duct 转发剩余数据
- 地址解析失败时发送错误状态并 FIN

**签名**

```cpp
auto activate_stream(std::uint32_t stream_id) -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`net::awaitable<void>` — 协程，连接完成或失败后返回。

**调用（向下）**

- `smux::parse_mux_address()` — 解析 SOCKS5 格式地址
- `router_.async_forward()` — 连接目标
- `channel::transport::make_reliable()` — 创建可靠传输层
- `make_duct()` — 创建 TCP 管道
- `make_parcel()` — 创建 UDP 管道
- `craft::send_data()` — 发送成功/错误状态
- `craft::send_fin()` — 连接失败时发送 FIN

**被调用（向上）**

- `craft::dispatch_push()` — 通过 co_spawn 启动

**知识域**

地址解析、TCP/UDP 分流、连接失败处理。

---

### craft::push_frame (私有)

**功能说明**

将帧推送到发送通道。编码帧头为 8 字节数组，与 payload 组装为 outbound_frame，通过 channel_ 发送通道传递给 send_loop。header 与 payload 分离，不执行 serialize 的 payload memcpy。

**签名**

```cpp
auto push_frame(command cmd, std::uint32_t stream_id, memory::vector<std::byte> payload) const
    -> net::awaitable<void>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `cmd` | `command` | 帧命令类型 |
| `stream_id` | `uint32_t` | 流标识符 |
| `payload` | `memory::vector<byte>` | 帧载荷（所有权转移） |

**返回值**

`net::awaitable<void>` — 协程，帧推入通道后完成。

**调用（向下）**

- 内部 `build_header()` — 编码帧头
- `channel_.async_send()` — 推入发送通道

**被调用（向上）**

- `craft::send_data()` — 发送 PSH 帧
- `craft::send_fin()` — 发送 FIN 帧
- `craft::keepalive_loop()` — 发送 NOP 帧

**知识域**

帧编码、有界通道发送。

---

### craft::send_loop (私有)

**功能说明**

发送循环协程，scatter-gather 写入 transport。从 channel_ 取出 outbound_frame，先写 8 字节帧头，再写 payload。写入失败时调用 close() 关闭整个会话。

**签名**

```cpp
auto send_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，通道关闭或写入失败时退出。

**调用（向下）**

- `channel_.async_receive()` — 从通道取帧
- `transport_->async_write_scatter()` — scatter-gather 写入
- `transport_->async_write()` — 纯帧头写入
- `core::close()` — 写入失败时关闭

**被调用（向上）**

- `craft::run()` — 通过 co_spawn 启动

**知识域**

scatter-gather I/O、零拷贝写入。

---

### craft::keepalive_loop (私有)

**功能说明**

NOP 心跳循环。当 keepalive_interval_ms > 0 时运行，按配置间隔发送 NOP 帧保持连接活性。定时器等待期间会话关闭则退出。

**签名**

```cpp
auto keepalive_loop() -> net::awaitable<void>;
```

**参数**

无。

**返回值**

`net::awaitable<void>` — 协程，会话关闭时退出。

**调用（向下）**

- `steady_timer::async_wait()` — 等待心跳间隔
- `craft::push_frame()` — 发送 NOP 帧

**被调用（向上）**

- `craft::run()` — 通过 co_spawn 启动

**知识域**

心跳保活机制、定时器协程。

---

## 设计约束

- 通过 core 的虚函数接口发送帧，不依赖具体协议
- channel_ 有界发送通道串行化多流写入
- header 与 payload 分离传递，零拷贝
- dispatch_push 使用 detached 协程避免慢速 target 阻塞帧循环
