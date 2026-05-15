---
title: Multiplex 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [multiplex, smux, yamux, frame, stream, overview]
related: [[channel], [pipeline/overview], [protocol/overview], [resolve], [memory/overview], [fault/overview]]
---

# Multiplex 模块

## 模块定位

Multiplex 是 Prism 的多路复用层，位于 [[channel]] 传输通道之上、
[[pipeline]] 管道层之下。它在单个 TCP/TLS 连接上承载多条独立的
双向流，实现连接复用，减少握手开销。

模块支持两种多路复用协议：smux（兼容 Mihomo/xtaci/smux v1）和
yamux（兼容 Hashicorp/yamux）。客户端通过 sing-mux 协商选择协议。

## 核心类

### core（协议无关抽象基类）

`multiplex/core.hpp` + `.cpp`。继承 `enable_shared_from_this<core>`，
管理流生命周期和发送串行化。子类 `smux::craft` 和 `yamux::craft`
实现协议特定的帧格式、解析和协商。

core 管理三种流状态映射：
- `pending_`：SYN 后等待地址数据的流（`pending_entry` 累积 PSH 帧数据）
- `ducts_`：已连接的 TCP 流管道（`shared_ptr<duct>`）
- `parcels_`：活跃的 UDP 数据报管道（`shared_ptr<parcel>`）

纯虚接口：`send_data`、`send_fin`、`executor`、`run`。

### sing-mux 协商（bootstrap）

`bootstrap()` 是多路复用会话的统一入口。从 transport 读取 sing-mux
协议头并根据 Protocol 字段（0=smux, 1=yamux）创建对应 core 子类。

协商格式：
- 基本：`[Version 1B][Protocol 1B]`
- 扩展（Version>0）：`[Version 1B][Protocol 1B][PaddingLen 2B BE][Padding]`

### duct（TCP 流管道）

`multiplex/duct.hpp` + `.cpp`。协议无关的双向 TCP 转发管道。
每条 duct 绑定一个已连接的 target 传输层。`start()` 启动两个独立协程：

- **下载方向**：`target_read_loop` 读 target 数据 → `core::send_data` 发到 mux
- **上传方向**：`on_mux_data` 接收 mux 数据 → `write_channel_` → `target_write_loop` 写 target

有界 `write_channel_`（`concurrent_channel`）提供反压。`mux_closed_` 和
`target_closed_` 原子标志实现半关闭语义。`owner_` 使用 `weak_ptr<core>`
避免循环引用。

### parcel（UDP 数据报管道）

`multiplex/parcel.hpp` + `.cpp`。协议无关的 UDP 数据报中继管道。

与 duct 的流模型不同，parcel 是请求-响应模型：
- `on_mux_data` 累积数据到 `mux_buffer_`，由 `process_buffer` 流式解析
- `do_send` 执行 DNS 解析 + UDP 发送，首次发送时启动 `downlink_loop`
- `downlink_loop` 持续读取 UDP 响应，编码后通过 `core::send_data` 回传

支持两种数据格式：PacketAddr（每帧带 SOCKS5 地址）和 length-prefixed。
空闲超时由 `idle_timer_` 管理，超时自动 `close()`。

## 配置

`multiplex::config` 聚合全局开关和各协议独立配置：
- `bool enabled`：是否启用多路复用
- `smux::config smux`：smux 配置（max_streams、buffer_size、keepalive 等）
- `yamux::config yamux`：yamux 配置（max_streams、initial_window、enable_ping 等）

## 设计原则

- **协议无关**：`core`/`duct`/`parcel` 不依赖具体帧格式
- **协程架构**：所有异步操作返回 `net::awaitable<T>`
- **反压机制**：有界 `concurrent_channel` 防止内存膨胀
- **零拷贝发送**：`outbound_frame` 的 header 与 payload 分离传递
- **生命周期安全**：`shared_from_this` + `weak_ptr` 防止循环引用

## 核心类实现细节

### core（抽象基类）

`multiplex/core.hpp` + `.cpp`。继承 `enable_shared_from_this`。

```
class core:
  friend duct, parcel
  core(shared_transmission transport, router&, const config&, resource_pointer)
  void start()              // co_spawn run()，异常/退出自动 close()
  virtual void close()      // 幂等：exchange(false) → cancel → 清空所有流
  bool is_active()          // atomic acquire
  virtual send_data(stream_id, vector<byte>) -> awaitable<void> = 0
  virtual send_fin(stream_id) = 0
  virtual executor() -> any_io_executor = 0
protected:
  virtual remove_duct / remove_parcel
  pending_entry { vector<byte> buffer; bool connecting; }
  unordered_map<uint32, pending_entry> pending_
  unordered_map<uint32, shared_ptr<duct>> ducts_
  unordered_map<uint32, shared_ptr<parcel>> parcels_
```

流状态：SYN → pending_entry（≥7 字节触发 activate_stream）→ 解析 SOCKS5 地址
→ router 连接目标 → 发送 0x00/0x01 状态 → 创建 duct/parcel。

### duct 实现细节

`enable_shared_from_this`。

```
class duct:
  duct(stream_id, shared_ptr<core> owner, shared_transmission target, buffer_size, mr)
  void start()                              // co_spawn 两个独立协程
  awaitable<void> on_mux_data(vector<byte>) // 推入 write_channel_（有界反压，容量 32）
  void on_mux_fin()                         // shutdown_write + 检查双端关闭
  void close()                              // 幂等
  weak_ptr<core> owner_; shared_transmission target_
  uint32 read_size_ = min(buffer_size, 65535)
  atomic<bool> mux_closed_, target_closed_
  concurrent_channel write_channel_
```

半关闭：mux FIN → shutdown target write；target EOF → send_fin。两端均关闭后析构。

### parcel 实现细节

`enable_shared_from_this`。

```
class parcel:
  parcel(stream_id, shared_ptr<core> owner, router&, timeout, max_dg, mr, packet_addr)
  void start()                              // 启动 idle_timer
  awaitable<void> on_mux_data(span<byte>)   // 累积 → process_buffer
  void close()                              // 幂等
  uplink_loop()        // idle_timer 超时 → close()
  process_buffer()     // swap 缓冲区 → 流式解析 → do_send
  do_send()            // DNS + send_to（首次启动 downlink_loop）
  downlink_loop()      // recv_from → 编码回传 → send_data
  ensure_socket()      // 延迟创建/按协议族重建
  bool packet_addr_; optional<udp::socket> egress_socket_
  vector<byte> mux_buffer_; size_t mux_offset_
  atomic<bool> processing_
```

### bootstrap 实现细节

`bootstrap(transport, router, config, mr)` → `awaitable<shared_ptr<core>>`。
读取 `[Version 1B][Protocol 1B]`（Version>0 读 padding），
Protocol 0 创建 `smux::craft`，1 创建 `yamux::craft`。

### config 实现细节

```
enum protocol_type: uint8 { smux=0, yamux=1 }
struct config { bool enabled; smux::config smux; yamux::config yamux; }
```

### smux 帧格式

`multiplex/smux/frame.hpp` + `.cpp`。

```
帧头 8B: [Version 1B][Cmd 1B][Length 2B LE][StreamID 4B LE]
enum command: syn=0, fin=1, push=2, nop=3
protocol_version=0x01, max_frame_length=65535
```

结构体：`frame_header`、`parsed_address`（sing-mux StreamRequest）、
`udp_datagram`（SOCKS5 格式）、`udp_length_prefixed`。
函数：`deserialization()`、`parse_mux_address()`、`parse_udp_datagram()`、
`parse_udp_length_prefixed()`、`build_udp_datagram()`、`build_udp_length_prefixed()`。

### smux::craft（会话服务端）

`multiplex/smux/craft.hpp` + `.cpp`。`final : public core`。

```
run()                       // send_loop + keepalive_loop → frame_loop
frame_loop()                // 读 8B 帧头 + payload，按 cmd 分发
handle_syn(stream_id)       // max_streams 检查 → pending_entry
dispatch_push(stream_id, payload) // 三路：pending 累积 / duct co_spawn / parcel co_spawn
handle_fin(stream_id)       // pending→erase / duct→on_mux_fin / parcel→close
activate_stream(stream_id)  // 解析地址 → TCP: router+duct / UDP: parcel
push_frame(cmd, id, payload)// 编码帧头 → outbound_frame → channel_
send_loop()                 // channel_ → scatter-gather（零拷贝）
keepalive_loop()            // NOP 心跳
```

配置：`max_streams=32`、`buffer_size=4096`、`keepalive_interval_ms=30000`。

### yamux 帧格式

`multiplex/yamux/frame.hpp` + `.cpp`。

```
帧头 12B: [Version 1B][Type 1B][Flags 2B BE][StreamID 4B BE][Length 4B BE]
Version=0。Type: data(0)/window_update(1)/ping(2)/go_away(3)。
Flags: syn/ack/fin/rst，可组合。initial_stream_window=256KB
```

函数：`build_header()`、`parse_header()`、`build_window_update_frame()`、
`build_ping_frame()`、`build_go_away_frame()`、`make_data_frame()`、
`make_syn_frame()`、`make_fin_frame()`。

### yamux::craft（会话服务端）

`multiplex/yamux/craft.hpp` + `.cpp`。`final : public core`。

```
run()                               // send_loop + ping_loop → frame_loop
frame_loop()                        // 读 12B 帧头，按 type 分发
handle_data(hdr, payload)           // SYN/RST/FIN/无标志 → dispatch_data
handle_syn(stream_id, payload)      // pending + WindowUpdate(ACK) + 超时 + 激活
handle_rst / handle_fin             // 清理 pending/duct/parcel + 窗口
dispatch_data(stream_id, payload)   // 三路分发 + 更新接收窗口
handle_window_update(hdr)           // RST/FIN/SYN/普通窗口更新 + 唤醒发送方
handle_ping(hdr) / handle_go_away(hdr)
activate_stream(stream_id)  // 解析地址 → 连接目标
push_frame / send_loop / ping_loop()
windows_: unordered_map<uint32, unique_ptr<stream_window>>
  stream_window { atomic send_window; atomic recv_consumed; shared_ptr<steady_timer> }
pending_timers_: 超时定时器
```

窗口管理：`send_window` 由 WindowUpdate 增加，发送前检查并等待；
`recv_consumed` 达 `initial_stream_window/2` 阈值时发送 WindowUpdate。

配置：`max_streams=32`、`initial_window=256KB`、`enable_ping=true`、
`ping_interval_ms=30000`、`stream_open/close_timeout_ms=30000`。

## 文件清单

| 文件 | 说明 |
|------|------|
| `multiplex/core.hpp` / `.cpp` | 核心抽象基类 |
| `multiplex/duct.hpp` / `.cpp` | TCP 流管道 |
| `multiplex/parcel.hpp` / `.cpp` | UDP 数据报管道 |
| `multiplex/bootstrap.hpp` / `.cpp` | sing-mux 协商入口 |
| `multiplex/config.hpp` | 通用配置 |
| `multiplex/smux/frame.hpp` / `.cpp` | smux 帧格式与编解码 |
| `multiplex/smux/craft.hpp` / `.cpp` | smux 会话服务端 |
| `multiplex/smux/config.hpp` | smux 配置 |
| `multiplex/yamux/frame.hpp` / `.cpp` | yamux 帧格式与编解码 |
| `multiplex/yamux/craft.hpp` / `.cpp` | yamux 会话服务端 |
| `multiplex/yamux/config.hpp` | yamux 配置 |
| `multiplex.hpp` | 聚合头文件 |

## 与其他模块的关系

- **上游**：[[pipeline]] 通过 `bootstrap()` 创建 mux 会话
- **下游**：使用 [[channel]] 的 `shared_transmission` 作为底层传输
- **下游**：使用 [[resolve]] 的 `router` 进行地址解析和目标连接
- **配置**：`multiplex::config` 聚合 `smux::config` 和 `yamux::config`
