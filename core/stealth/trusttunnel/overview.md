---
title: TrustTunnel 协议
created: 2026-05-27
updated: 2026-05-29
layer: core
source:
  - include/prism/stealth/stack/trusttunnel/scheme.hpp
  - include/prism/stealth/stack/trusttunnel/config.hpp
  - src/prism/stealth/stack/trusttunnel/scheme.cpp
tags: [stealth, trusttunnel, overview, h2, http2-connect, basic-auth]
---

# TrustTunnel 协议

> 源码: `include/prism/stealth/stack/trusttunnel/` | 实现: `src/prism/stealth/stack/trusttunnel/`

TrustTunnel 是 Stealth 模块中 Tier 2 级 stack 类型伪装方案，通过 HTTP/2 CONNECT 代理协议建立安全隧道。使用 Basic Auth 认证客户端，通过 h2mux（nghttp2）实现 HTTP/2 多路复用，仅首条 CONNECT 流需要认证。位于 `stealth/stack/` 而非 `facade/`，因为它内部管理流而非返回 transport。

## 模块架构

```
TLS 连接到达（ALPN 协商 h2）
       |
       v
+----------------------------------------------------------+
|  scheme::handshake()                                     |
|                                                          |
|  1. ALPN 检查                                            |
|     协商 h2 -> 继续                                      |
|     非 h2 -> fallback 到原生 TLS                         |
|                                                          |
|  2. h2mux 初始化                                         |
|     创建 h2mux::craft 实例（nghttp2 session）            |
|     将 TLS 传输层交给 h2mux 管理                         |
|                                                          |
|  3. 等待首条 CONNECT 流                                  |
|     resolve_stream_target() 解析 CONNECT 请求            |
|     验证目标地址合法性                                    |
|                                                          |
|  4. Basic Auth 验证                                      |
|     verify_basic_auth() 检查 Authorization 头            |
|     用户名:密码 与配置列表匹配                           |
|     认证失败 -> 返回 407 Proxy Authentication Required   |
|                                                          |
|  5. 建立代理隧道                                        |
|     首条流: dial 目标 -> 双向转发                        |
|     后续流: 无需认证，直接代理                           |
+----------------------------------------------------------+
```

### 核心组件

| 组件 | 头文件 | 职责 |
|------|--------|------|
| [[core/stealth/trusttunnel/scheme\|scheme]] | `stack/trusttunnel/scheme.hpp` | 方案注册、handshake 入口、ALPN 检查 |
| [[core/stealth/trusttunnel/config\|config]] | `stack/trusttunnel/config.hpp` | 用户列表配置、BBR 拥塞控制默认值 |

## 子页面

- [[core/stealth/trusttunnel/scheme|Scheme]] — 方案基类实现
- [[core/stealth/trusttunnel/config|Config]] — 配置参数

## 设计决策（WHY）

### 为什么用 HTTP/2 CONNECT 而非自定义协议？

**问题**: 需要一个 DPI 友好的代理协议，流量特征应与正常 HTTPS 一致。

**选择**: 使用标准 HTTP/2 CONNECT 方法建立隧道。HTTP/2 CONNECT 是 RFC 8441 定义的标准代理机制，被 CDN 和云服务广泛使用。DPI 看到的是正常的 h2 流量，无法区分代理隧道。

**后果**: 需要完整的 HTTP/2 实现（nghttp2），增加依赖。ALPN 必须协商 h2，否则无法工作。

### 为什么只需首条流认证？

**问题**: HTTP/2 多路复用允许在同一连接上开多个流，每个流都需要认证吗？

**选择**: 仅首条 CONNECT 流要求 Authorization 头。后续流直接代理，无需重复认证。原因是：HTTP/2 连接建立时 TLS 已完成握手，连接级别已绑定到客户端。每条流都认证会增加延迟且无安全收益——攻击者如果能开新流，说明已经持有了已认证的连接。

**后果**: 连接劫持（中间人接管已建立的 h2 连接）可绕过流级认证。但 TLS 层已防劫持。

### 为什么是 stack 类型而非 facade？

**问题**: 其他伪装方案（Reality、ShadowTLS）是 facade 类型，返回 transport + preread。TrustTunnel 为什么不同？

**选择**: TrustTunnel 使用 h2mux 在内部管理 HTTP/2 帧和多路复用流，session 层不需要（也不能）直接操作底层 transport。h2mux 本身就是传输层，内部处理流的多路复用。因此 TrustTunnel 是 stack 类型——它内部管理完整的流生命周期。

**后果**: TrustTunnel 不能与 facade 类型方案（Reality、ShadowTLS）嵌套使用。它独立管理从 TLS 到代理转发的完整链路。

### 为什么默认使用 BBR 拥塞控制？

**问题**: h2mux 内部的流控策略影响代理性能。

**选择**: 配置中 BBR 作为默认拥塞控制算法。BBR 基于带宽和 RTT 模型，在高延迟、有丢包的网络环境下表现优于 Cubic（默认 TCP 拥塞控制）。代理流量通常穿越长距离网络，BBR 更适合。

**后果**: BBR 在低带宽共享链路上可能不公平地占用带宽。对于代理场景这通常可接受。

## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| ALPN 必须协商 h2 | TrustTunnel 依赖 HTTP/2 帧格式 | 客户端协商非 h2 时 fallback 到原生 TLS | `scheme.cpp` |
| 首条流必须带 Authorization | Basic Auth 认证 | 缺少 Authorization 返回 407，连接保留 | `scheme.cpp` |
| h2mux 管理传输层 | stack 类型，不返回 transport | session 层不能直接操作底层 socket | `scheme.hpp` |
| 用户列表为空则禁用 | config.enabled() 检查 users 非空 | 无法启用 TrustTunnel | `config.hpp` |
| CONNECT 目标合法性 | resolve_stream_target() 验证 | 非法目标拒绝代理 | `scheme.cpp` |

## 故障场景

### 1. ALPN 协商非 h2

**触发条件**: 客户端 TLS 握手时 ALPN 协商结果不是 h2

**传播路径**: handshake() 检查 ALPN -> 非 h2 -> 将传输层交给下一个方案处理

**外部表现**: 连接被当作普通 TLS 处理

### 2. Basic Auth 认证失败

**触发条件**: 首条 CONNECT 请求的 Authorization 头不匹配配置的用户列表

**传播路径**: verify_basic_auth() -> 用户名或密码不匹配 -> 返回 407

**外部表现**: 客户端收到 407 Proxy Authentication Required

**恢复机制**: 客户端可重试带正确凭证的请求

### 3. nghttp2 会话错误

**触发条件**: h2mux 内部 nghttp2 会话遇到帧解析错误

**传播路径**: h2mux::craft 报错 -> TrustTunnel 关闭连接

**外部表现**: 连接中断

### 4. CONNECT 目标不可达

**触发条件**: CONNECT 请求指定的目标地址无法建立 TCP 连接

**传播路径**: dial 目标失败 -> 返回 502 Bad Gateway 给客户端

**外部表现**: 客户端收到 502 响应

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| TrustTunnel -> [[core/multiplex/h2mux/craft\|h2mux]] | 依赖 | 使用 h2mux::craft（nghttp2）处理 HTTP/2 帧和多路复用 |
| TrustTunnel -> [[core/transport/transmission\|transport]] | 依赖 | TLS 传输层作为 h2mux 的底层传输 |
| TrustTunnel -> [[core/connect/dial/dial\|connect]] | 依赖 | dial 建立 TCP 连接到 CONNECT 目标 |
| TrustTunnel -> [[core/instance/worker/tls\|instance]] | 依赖 | worker 的 tls::configure() 提供 TLS 上下文 |
| [[core/instance/overview\|instance]] <- TrustTunnel | 被依赖 | session 通过 scheme::handshake() 调用 TrustTunnel |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| 用户列表变更 | 所有客户端 | 旧凭证失效，新连接使用新凭证 |
| BBR 配置变更 | 性能 | 流控行为变化，影响吞吐和延迟 |
| h2 ALPN 要求 | 客户端兼容性 | 不支持 h2 的客户端无法使用 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| h2mux::craft API 变更 | handshake() 的 h2mux 初始化和使用 | 流回调、帧读写接口 |
| nghttp2 库升级 | h2mux 内部 nghttp2 调用 | 帧处理回调签名 |
| connect::dial 接口变更 | CONNECT 目标连接建立 | 错误处理和超时 |
| transport::transmission 接口变更 | TLS 传输层与 h2mux 的衔接 | async_read/write 签名 |

## 相关文档

- [[core/stealth/overview|Stealth 模块总览]] — 三级检测架构和方案执行器
- [[core/multiplex/h2mux/craft|h2mux]] — HTTP/2 多路复用实现
- [[core/connect/dial/dial|Dial]] — 目标连接建立
- [[core/transport/transmission|Transport]] — 传输层抽象接口
