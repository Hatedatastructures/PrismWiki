---
title: "config — SS2022 协议配置"
source: "include/prism/protocol/shadowsocks/config.hpp"
module: "protocol"
type: api
tags: [protocol, shadowsocks, config, 配置, PSK, aead, timestamp]
related:
  - "[[protocol/shadowsocks/constants|constants]]"
  - "[[protocol/shadowsocks/relay|relay]]"
  - "[[agent/config|agent_config]]"
created: 2026-05-15
updated: 2026-05-15
---

# config.hpp

> 源码: `include/prism/protocol/shadowsocks/config.hpp`
> 模块: [[protocol|protocol]] > shadowsocks

## 概述

SS2022 协议配置。定义 SS2022 协议的运行时配置结构，包括 PSK、加密方法选择、TCP/UDP 开关和时间戳窗口参数。PSK 以 Base64 编码存储，运行时解码为原始字节。解码后必须为 16 字节（AES-128）或 32 字节（AES-256/ChaCha20）。`method` 字段用于区分 32 字节 PSK 对应的加密算法。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory|memory]] | `memory::string` |
| 被依赖 | [[protocol/shadowsocks/relay|relay]] | 中继器构造时传入配置 |
| 被依赖 | [[agent/config|agent_config]] | 服务启动时初始化配置 |

## 命名空间

`psm::protocol::shadowsocks`

---

## 结构体: config

### 功能说明

SS2022 协议配置。配置由 `agent::config` 持有，随服务启动时初始化。PSK 运行时通过 `format::decode_psk()` 解码。

### 字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `psk` | `memory::string` | 空 | Pre-Shared Key（Base64 编码）。解码后为 16 或 32 字节 |
| `method` | `memory::string` | 空 | 加密方法名（可选，自动推断：16B -> aes-128-gcm，32B -> aes-256-gcm）。显式设置时支持 `"2022-blake3-chacha20-poly1305"` |
| `enable_tcp` | `bool` | `true` | 是否启用 TCP 代理 |
| `enable_udp` | `bool` | `false` | 是否启用 UDP 代理，默认禁用 |
| `timestamp_window` | `std::int64_t` | `30` | 时间戳防重放窗口（秒）。客户端时间戳与服务端时间差超过此值则拒绝 |
| `salt_pool_ttl` | `std::int64_t` | `60` | Salt 池 TTL（秒）。每个 salt 在此时间内只能出现一次 |
| `udp_idle_timeout` | `std::uint32_t` | `60` | UDP 会话空闲超时（秒） |

### 配置示例

```json
{
  "psk": "base64encodedkey==",
  "method": "2022-blake3-aes-256-gcm",
  "enable_tcp": true,
  "enable_udp": false,
  "timestamp_window": 30,
  "salt_pool_ttl": 60,
  "udp_idle_timeout": 60
}
```

## 相关页面

- [[protocol/shadowsocks/constants|constants]] — 协议常量
- [[protocol/shadowsocks/format|format]] — 格式编解码
- [[protocol/shadowsocks/relay|relay]] — 协议中继器
- [[agent/config|agent_config]] — Agent 全局配置
