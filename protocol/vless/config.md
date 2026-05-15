---
title: "config — VLESS 协议配置"
source: "include/prism/protocol/vless/config.hpp"
module: "protocol"
type: api
tags: [protocol, vless, config, 配置, UDP]
related:
  - "[[protocol/vless/constants|constants]]"
  - "[[protocol/vless/relay|relay]]"
  - "[[agent/config|agent_config]]"
created: 2026-05-15
updated: 2026-05-15
---

# config.hpp

> 源码: `include/prism/protocol/vless/config.hpp`
> 模块: [[protocol|protocol]] > vless

## 概述

VLESS 协议配置。用户认证已统一到 `agent::authentication`，VLESS 通过 `account::directory` 查询 UUID 凭证。配置由 `agent::config` 持有，随服务启动时初始化。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/vless/relay|relay]] | 中继器构造时传入配置 |
| 被依赖 | [[agent/config|agent_config]] | 服务启动时初始化配置 |

## 命名空间

`psm::protocol::vless`

---

## 结构体: config

### 功能说明

控制 VLESS 协议的能力开关和 UDP 参数。配置在 relay 构造时传入，运行时不可修改。

### 字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enable_udp` | `bool` | `false` | 是否允许 UDP 命令（UDP over TLS），默认禁用 |
| `udp_idle_timeout` | `std::uint32_t` | `60` | UDP 会话空闲超时时间（秒） |
| `udp_max_datagram` | `std::uint32_t` | `65535` | UDP 数据报最大长度（字节） |

## 相关页面

- [[protocol/vless/constants|constants]] — 协议常量
- [[protocol/vless/relay|relay]] — 协议中继器
- [[agent/config|agent_config]] — Agent 全局配置
