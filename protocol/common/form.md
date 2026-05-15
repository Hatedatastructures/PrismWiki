---
title: "form — 传输形态枚举"
source: "include/prism/protocol/common/form.hpp"
module: "protocol"
type: api
tags: [protocol, common, form, stream, datagram, TCP, UDP]
related:
  - "[[protocol/socks5/constants|socks5_constants]]"
  - "[[protocol/trojan/constants|trojan_constants]]"
  - "[[pipeline/pipeline|pipeline]]"
created: 2026-05-15
updated: 2026-05-15
---

# form.hpp

> 源码: `include/prism/protocol/common/form.hpp`
> 模块: [[protocol|protocol]] > common

## 概述

传输形态枚举定义。定义了传输层的数据传输形态，用于区分 TCP 流式传输和 UDP 数据报传输。该枚举在协议层用于标识请求的传输形态，指导 pipeline 进行正确的路由分发。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/socks5/message|socks5_message]] | SOCKS5 请求使用 form |
| 被依赖 | [[protocol/trojan/message|trojan_message]] | Trojan 请求使用 form |
| 被依赖 | [[protocol/vless/message|vless_message]] | VLESS 请求使用 form |
| 被依赖 | [[pipeline/pipeline|pipeline]] | 根据 form 选择 TCP tunnel 或 UDP relay |

## 命名空间

`psm::protocol`

---

## 枚举: form

标识数据传输的形态，用于协议层和 pipeline 层的路由决策。

| 值 | 数值 | 说明 |
|----|------|------|
| `stream` | `0` | TCP 可靠流传输，保证可靠性和顺序 |
| `datagram` | `1` | UDP 数据报传输，不保证可靠性 |

### 映射关系

| 命令 | form | 说明 |
|------|------|------|
| SOCKS5 CONNECT | `stream` | TCP 隧道 |
| SOCKS5 UDP_ASSOCIATE | `datagram` | UDP 中继 |
| SOCKS5 BIND | `stream` | 控制面 |
| Trojan CONNECT | `stream` | TCP 隧道 |
| Trojan UDP_ASSOCIATE | `datagram` | UDP over TLS |
| VLESS TCP | `stream` | TCP 代理 |
| VLESS UDP | `datagram` | UDP 代理 |

## 相关页面

- [[protocol/socks5/constants|socks5_constants]] — SOCKS5 命令字
- [[protocol/trojan/constants|trojan_constants]] — Trojan 命令字
- [[pipeline/pipeline|pipeline]] — 协议处理管道
