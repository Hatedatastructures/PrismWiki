---
title: "config — smux 协议配置"
source: "include/prism/multiplex/smux/config.hpp"
module: "multiplex"
type: api
tags: [multiplex, smux, config, 配置]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/config|config]]"
  - "[[multiplex/smux/craft|smux::craft]]"
  - "[[multiplex/core|core]]"
  - "[[multiplex/duct|duct]]"
  - "[[multiplex/parcel|parcel]]"
---

# config.hpp — smux 协议配置

> 源码: `include/prism/multiplex/smux/config.hpp`
> 模块: [[multiplex/config|multiplex]] > smux

## 概述

smux 协议配置。定义 smux 协议的全部配置参数，包括流数量限制、读取缓冲区、保活心跳和 UDP 中继参数。作为 [[multiplex/config|multiplex::config]] 的 smux 子配置存在，由 agent 配置统一加载。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[multiplex/config|config]] | 嵌入为 smux 子配置 |
| 被依赖 | [[multiplex/smux/craft|smux::craft]] | 读取各配置参数控制行为 |
| 被依赖 | [[multiplex/core|core]] | 通过 config 引用间接使用 |

## 命名空间

`psm::multiplex::smux`

---

## 结构体: config

**功能说明**

smux 协议配置。控制 smux 服务端行为的完整参数集。max_streams 限制单会话并发流数，buffer_size 控制每流 target 读取的单次数据量上限（不超过帧最大载荷 65535），keepalive_interval_ms 控制 NOP 心跳帧发送间隔。

**签名**

```cpp
struct config {
    std::uint32_t max_streams = 32;
    std::uint32_t buffer_size = 4096;
    std::uint32_t keepalive_interval_ms = 30000;
    std::uint32_t udp_idle_timeout_ms = 60000;
    std::uint32_t udp_max_datagram = 65535;
};
```

**字段**

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_streams` | `uint32_t` | `32` | 单个 mux 会话最大并发流数 |
| `buffer_size` | `uint32_t` | `4096` | 每流读取缓冲区大小（字节），实际限制为 min(buffer_size, 65535) |
| `keepalive_interval_ms` | `uint32_t` | `30000` | 心跳间隔（毫秒），0 表示禁用心跳 |
| `udp_idle_timeout_ms` | `uint32_t` | `60000` | UDP 管道空闲超时（毫秒），超时自动关闭 |
| `udp_max_datagram` | `uint32_t` | `65535` | UDP 数据报最大长度（字节） |

**返回值**

无（结构体定义）。

**调用（向下）**

无。

**被调用（向上）**

- `smux::craft::craft()` — 使用 max_streams 初始化 channel_ 容量
- `smux::craft::handle_syn()` — 使用 max_streams 检查流数量限制
- `smux::craft::frame_loop()` — 使用 buffer_size 读取帧载荷
- `smux::craft::keepalive_loop()` — 使用 keepalive_interval_ms 控制心跳间隔
- `smux::craft::activate_stream()` — 使用 udp_idle_timeout_ms 和 udp_max_datagram 创建 parcel
- `duct` 构造函数 — 使用 buffer_size 计算 read_size_

**知识域**

配置参数语义、流数量限制、心跳保活、UDP 超时管理。
