---
title: "config — yamux 协议配置"
source: "include/prism/multiplex/yamux/config.hpp"
module: "multiplex"
type: api
tags: [multiplex, yamux, config, 配置, 流量控制]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/config|config]]"
  - "[[multiplex/yamux/craft|yamux::craft]]"
  - "[[multiplex/core|core]]"
  - "[[multiplex/duct|duct]]"
  - "[[multiplex/parcel|parcel]]"
---

# config.hpp — yamux 协议配置

> 源码: `include/prism/multiplex/yamux/config.hpp`
> 模块: [[multiplex/config|multiplex]] > yamux

## 概述

yamux 协议配置。定义 yamux 协议的全部配置参数，包括流数量限制、流量控制窗口、心跳和超时设置以及 UDP 中继参数。作为 [[multiplex/config|multiplex::config]] 的 yamux 子配置存在，由 agent 配置统一加载。

initial_window 影响单流吞吐量，增大可提升高延迟链路的传输效率。enable_ping 和 ping_interval_ms 控制 Ping 心跳行为，stream_open/close_timeout_ms 控制流生命周期超时。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[multiplex/config|config]] | 嵌入为 yamux 子配置 |
| 被依赖 | [[multiplex/yamux/craft|yamux::craft]] | 读取各配置参数控制行为 |
| 被依赖 | [[multiplex/core|core]] | 通过 config 引用间接使用 |

## 命名空间

`psm::multiplex::yamux`

---

## 结构体: config

**功能说明**

yamux 协议配置。控制 yamux 服务端行为的完整参数集。

**签名**

```cpp
struct config {
    std::uint32_t max_streams = 32;
    std::uint32_t buffer_size = 4096;
    std::uint32_t initial_window = 256 * 1024;
    bool enable_ping = true;
    std::uint32_t ping_interval_ms = 30000;
    std::uint32_t stream_open_timeout_ms = 30000;
    std::uint32_t stream_close_timeout_ms = 30000;
    std::uint32_t udp_idle_timeout_ms = 60000;
    std::uint32_t udp_max_datagram = 65535;
};
```

**字段**

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_streams` | `uint32_t` | `32` | 单个 mux 会话最大并发流数 |
| `buffer_size` | `uint32_t` | `4096` | 每流读取缓冲区大小（字节） |
| `initial_window` | `uint32_t` | `262144` (256KB) | 初始流窗口大小（字节），控制单流发送量 |
| `enable_ping` | `bool` | `true` | 是否启用心跳 |
| `ping_interval_ms` | `uint32_t` | `30000` | 心跳间隔（毫秒） |
| `stream_open_timeout_ms` | `uint32_t` | `30000` | 流打开超时（毫秒），超时后发送 RST |
| `stream_close_timeout_ms` | `uint32_t` | `30000` | 流关闭超时（毫秒） |
| `udp_idle_timeout_ms` | `uint32_t` | `60000` | UDP 管道空闲超时（毫秒），超时自动关闭 |
| `udp_max_datagram` | `uint32_t` | `65535` | UDP 数据报最大长度（字节） |

**返回值**

无（结构体定义）。

**调用（向下）**

无。

**被调用（向上）**

- `yamux::craft::craft()` — 使用 max_streams 初始化 channel_ 容量
- `yamux::craft::handle_syn()` — 使用 max_streams 检查流数量限制、initial_window 发送 ACK
- `yamux::craft::handle_window_update()` — 使用 initial_window 初始化发送窗口
- `yamux::craft::frame_loop()` — 使用 buffer_size 读取帧载荷
- `yamux::craft::run()` — 使用 enable_ping 和 ping_interval_ms 控制心跳
- `yamux::craft::ping_loop()` — 使用 ping_interval_ms 控制心跳间隔
- `yamux::craft::start_pending_timeout()` — 使用 stream_open_timeout_ms 控制超时
- `yamux::craft::activate_stream()` — 使用 udp_idle_timeout_ms 和 udp_max_datagram 创建 parcel
- `yamux::craft::update_recv_window()` — 使用 initial_window 计算阈值
- `duct` 构造函数 — 使用 buffer_size 计算 read_size_

**知识域**

流量控制参数语义、窗口大小与吞吐量关系、心跳保活、流超时管理。
