---
title: SS2022 协议处理入口
layer: core
source:
  - include/prism/protocol/shadowsocks/process.hpp
  - src/prism/protocol/shadowsocks/process.cpp
  - include/prism/protocol/shadowsocks/packet.hpp
tags: [protocol, shadowsocks, process]
---

# SS2022 协议处理入口

Shadowsocks 2022 协议的顶层编排函数，协调传输层包装、握手、确认和隧道转发的完整流程。

## 模块概述

`process` 模块声明唯一的入口函数 `handle()`，将 SS2022 协议处理拆解为四个阶段：

```
wrap_with_preview（预读数据重放）
  → conn::handshake（解密请求、验证时间戳、解析地址）
  → conn::acknowledge（发送服务端响应）
  → connect::forward（拨号 + 双向隧道转发）
```

`packet` 模块定义 `request` 结构体，由 `conn::handshake()` 填充后返回给 process 层。

## 核心函数

### handle()

```cpp
auto handle(context::session &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

**参数**：
- `ctx`：会话上下文，包含入站传输层、服务器配置、worker 上下文等
- `data`：recognition 阶段预读的字节数据（通过 preview 装饰器重放回传输流）

**处理流程**：

1. **传输层包装**：将预读数据通过 `transport::wrap_with_preview` 装饰到入站传输层上，确保握手时预读的字节被正确重放
2. **salt_pool 初始化**：通过 `thread_local` 为当前 worker 线程维护独立的 salt 重放保护池。TTL 变更时自动重建
3. **创建 conn**：调用 `make_conn` 构造 AEAD 中继器，传入入站传输层、SS2022 配置和 salt_pool
4. **握手**：`co_await agent->handshake()` 解密请求头、验证时间戳和 salt 唯一性、解析目标地址
5. **确认**：`co_await agent->acknowledge()` 发送服务端响应（乐观响应策略）
6. **转发**：将 conn 转型为 `transport::transmission`，调用 `connect::forward` 执行拨号和双向隧道转发

### request 结构体（packet.hpp）

```cpp
struct request
{
    cipher_method method;        // 加密算法
    std::uint16_t port{0};       // 目标端口
    address destination_address; // 目标地址
};
```

由 `conn::handshake()` 填充。地址类型复用 `protocol::common` 中的共享定义（`ipv4_address`、`ipv6_address`、`domain_address`），与 SOCKS5 地址格式兼容。

## 设计决策

### WHY: 乐观响应（先 acknowledge 再拨号）

- **Problem**：如果先拨号再响应，握手延迟 = RTT + 上游拨号时间，对高延迟上游影响显著。
- **Choice**：先发送 `acknowledge()` 再调用 `connect::forward` 拨号，与 mihomo 行为一致。
- **Consequence**：降低握手延迟；但拨号失败时客户端已收到成功响应，随后的连接重置对客户端而言是意外断开而非明确的连接拒绝。

### WHY: thread_local salt_pool

- **Problem**：salt 重放检测需要维护状态，多线程共享需加锁，与协程纯度原则冲突。
- **Choice**：每个 worker 线程持有独立的 `thread_local` salt_pool 实例，无锁访问。TTL 变更时（`cached_ttl != current_ttl`）自动重建。
- **Consequence**：零同步开销；跨线程的 salt 重复不会被检测，但由于 salt 为密码学随机值，冲突概率可忽略。

### WHY: preview 装饰器重放预读数据

- **Problem**：recognition 阶段已从传输层预读了若干字节（用于协议探测），这些字节必须回传给 SS2022 握手流程。
- **Choice**：使用 `transport::wrap_with_preview` 将预读数据装饰到传输层上，使 `conn` 的 `async_read_some` 先返回预读字节再从底层读取新数据。
- **Consequence**：SS2022 握手无需感知预读的存在，逻辑保持简洁。

## 约束

| 约束 | 说明 |
|------|------|
| 单入口 | 整个 SS2022 处理流程只有一个 `handle()` 函数，被 session 层分发调用 |
| salt_pool 线程绑定 | `thread_local` 生命周期与线程相同，worker 线程退出时自动释放 |
| conn 生命周期 | conn 作为 `shared_ptr` 传入 `connect::forward`，隧道结束时自动析构 |

## 失败场景

### 1. 握手失败

`conn::handshake()` 返回非 success 错误码（PSK 无效、salt 重放、时间戳过期、AEAD 解密失败、超时）。`handle()` 记录警告日志后直接返回，不发送任何响应——因为 SS2022 是无回显协议，握手失败时服务端直接断开连接。

### 2. 确认失败

`conn::acknowledge()` 写入响应失败（底层连接已断开）。记录警告日志后返回。此时客户端可能仍在等待响应，最终超时断开。

### 3. 上游拨号失败

`connect::forward` 内部拨号失败。由于已执行 `acknowledge()`，客户端已收到成功响应。forward 内部会处理错误日志和资源清理，客户端随后看到连接重置。

## 跨模块契约

| 上游模块 | 契约 |
|----------|------|
| [[core/protocol/shadowsocks/conn]] | 提供 `make_conn`、`handshake`、`acknowledge`、`target` |
| [[core/transport/preview]] | 提供 `wrap_with_preview` 装饰器 |
| [[core/connect/tunnel/forward]] | 提供 `connect::forward`，消费 conn 作为 inbound |
| [[core/context/context]] | 提供 `session` 上下文（inbound 传输层、配置、worker 上下文） |

## 变更敏感度

| 修改点 | 影响范围 |
|--------|----------|
| `handle()` 流程顺序 | 乐观响应策略依赖于 acknowledge 在 forward 之前，调换顺序会改变失败行为 |
| salt_pool TTL 配置位置 | 从 `ctx.server_ctx.config().protocol.shadowsocks.salt_ttl` 读取，修改配置键名需同步 |
| preview 装饰器接口 | `wrap_with_preview` 参数签名变更会影响所有协议的 process 层 |

## 相关文档

- [[core/protocol/shadowsocks/conn]] — AEAD 流加密中继器
- [[core/protocol/shadowsocks/constants]] — 协议常量
- [[core/protocol/shadowsocks/config]] — 配置结构
- [[core/protocol/shadowsocks/salts]] — Salt 重放保护池
- [[core/protocol/shadowsocks/format]] — 地址解析、PSK 解码
- [[core/transport/preview]] — 预读重放装饰器
- [[core/connect/tunnel/forward]] — 拨号 + 隧道转发
