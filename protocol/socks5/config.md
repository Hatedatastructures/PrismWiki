---
title: "config — SOCKS5 协议配置"
source: "include/prism/protocol/socks5/config.hpp"
module: "protocol"
type: api
tags: [protocol, socks5, config, 配置, UDP, 认证]
related:
  - "[[protocol/socks5/constants|constants]]"
  - "[[protocol/socks5/stream|stream]]"
  - "[[agent/config|agent_config]]"
created: 2026-05-15
updated: 2026-05-15
---

# config.hpp

> 源码: `include/prism/protocol/socks5/config.hpp`
> 模块: [[protocol|protocol]] > socks5

## 概述

SOCKS5 协议配置。控制 SOCKS5 协议的能力开关和 UDP relay 参数。配置由 `agent::config` 持有，随服务启动时初始化，并传递给 `socks5::stream` 构造函数，在 stream 生命周期内保持不变。配置结构本身是只读的，可安全多线程读取；修改配置需要重启服务。

默认值设计遵循最小权限原则：TCP 默认启用以满足最常用场景，UDP 默认启用以支持代理转发，BIND 默认禁用以降低安全风险。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/socks5/stream|stream]] | 中继器构造时传入配置 |
| 被依赖 | [[agent/config|agent_config]] | 服务启动时初始化配置 |

## 命名空间

`psm::protocol::socks5`

---

## 结构体: config

### 功能说明

控制 SOCKS5 协议的能力开关和 UDP 中继参数。配置在 stream 构造时传入，运行时不可修改。

### 字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enable_tcp` | `bool` | `true` | 是否允许 CONNECT 命令（TCP 隧道） |
| `enable_udp` | `bool` | `true` | 是否允许 UDP_ASSOCIATE 命令（UDP 中继） |
| `enable_bind` | `bool` | `false` | 是否允许 BIND 命令（通常禁用，降低安全风险） |
| `udp_bind_port` | `std::uint16_t` | `0` | UDP relay 绑定端口，0 表示由系统自动分配 |
| `udp_idle_timeout` | `std::uint32_t` | `60` | UDP 会话空闲超时（秒），超时后自动关闭关联 |
| `udp_max_datagram` | `std::uint32_t` | `65535` | UDP 数据报最大长度（字节） |
| `enable_auth` | `bool` | `false` | 是否启用用户名/密码认证（RFC 1929） |

### 使用场景

- `agent::config` 持有配置实例
- 服务启动时从 `configuration.json` 加载
- 传递给 `socks5::stream` 构造函数
- 运行期间只读

## 相关页面

- [[protocol/socks5/constants|constants]] — 协议常量
- [[protocol/socks5/stream|stream]] — SOCKS5 中继器
- [[agent/config|agent_config]] — Agent 全局配置
