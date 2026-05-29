---
title: HTTP 协议处理入口
layer: core
source:
  - I:/code/Prism/include/prism/protocol/http/process.hpp
  - I:/code/Prism/src/prism/protocol/http/process.cpp
tags: [protocol, http, process]
---

# HTTP 协议处理入口

HTTP 协议的顶层编排函数，协调传输层包装、握手认证、目标解析、拨号连接和隧道转发的完整流程。

## 模块概述

`process` 模块声明唯一的入口函数 `handle()`，将 HTTP 代理处理拆解为五个阶段：

```
wrap_with_preview（预读数据重放）
  → conn::handshake（读取请求头 + 解析 + 认证）
  → recognition::resolve（从 proxy_request 解析目标地址）
  → connect::dial（拨号连接目标服务器）
  → 按方法分发：
      CONNECT → conn::send_ok → tunnel
      其他    → conn::forward → tunnel
```

## 核心函数

### handle()

```cpp
auto handle(context::session &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

**参数**：
- `ctx`：会话上下文，包含入站传输层、账户目录、worker 上下文、出站代理等
- `data`：recognition 阶段预读的字节数据（通过 preview 装饰器重放回传输流）

**处理流程**：

1. **帧内存池重置**：`ctx.frame_arena.reset()`，为新会话提供干净的内存分配起点
2. **传输层包装**：将预读数据通过 `transport::wrap_with_preview` 装饰到入站传输层上
3. **握手**：创建 `conn` 并调用 `handshake()`，完成请求头读取、解析和 Basic 认证
4. **目标解析**：调用 `recognition::resolve(req)` 从解析后的 `proxy_request` 提取目标地址
5. **拨号连接**：支持两种模式——
   - 若配置了 `outbound_proxy`（正向代理），通过代理拨号
   - 否则通过 `ctx.worker_ctx.router` 直接拨号，使用 `dial_options::flag::no_open` 标志
6. **按方法分发**：
   - **CONNECT**：发送 `200 Connection Established` → 释放传输层 → 进入隧道
   - **其他 HTTP 方法**：调用 `forward` 重写 URI 并转发原始数据 → 释放传输层 → 进入隧道

**失败处理**：
- 握手失败：记录警告日志，直接返回
- 拨号失败：发送 `502 Bad Gateway` 响应后返回
- 响应写入失败：直接返回（连接将自然关闭）

## 设计决策

### WHY: CONNECT 与普通 HTTP 的分发

- **Problem**：HTTP 代理需要处理两种截然不同的场景——CONNECT 建立加密隧道，普通方法需要改写请求行后转发。
- **Choice**：在 `handle()` 中通过 `req.method == "CONNECT"` 进行分发。CONNECT 只需发送 200 响应后进入隧道；普通方法需先 `forward` 重写 URI 再进入隧道。
- **Consequence**：两条路径共享握手和拨号逻辑，仅在最后的响应和转发步骤分叉。代码简洁但需确保两条路径都正确调用 `release()`。

### WHY: 拨号前不发送响应

- **Problem**：如果先发送 200 再拨号，拨号失败时客户端已认为隧道建立。
- **Choice**：先拨号，拨号成功后再 `send_ok`（CONNECT）或 `forward`（普通 HTTP）。拨号失败时发送 `502 Bad Gateway`。
- **Consequence**：客户端能收到明确的错误响应；但握手延迟增加了一个 RTT（等待上游拨号完成）。这与 SS2022 的乐观响应策略不同。

### WHY: outbound_proxy 支持

- **Problem**：某些部署场景需要通过正向代理（如 HTTP/HTTPS 代理或 SOCKS5 代理）连接目标服务器。
- **Choice**：`handle()` 检查 `ctx.outbound_proxy` 是否存在，若存在则通过代理拨号，否则直接拨号。
- **Consequence**：HTTP 代理可以链式嵌套；但增加了拨号路径的复杂度，需要确保两条路径的错误处理一致。

## 约束

| 约束 | 说明 |
|------|------|
| 单入口 | 整个 HTTP 处理流程只有一个 `handle()` 函数 |
| 帧内存池 | 每次调用前必须重置 `ctx.frame_arena` |
| 传输层所有权 | `wrap_with_preview` 后 `ctx.inbound` 被置为 nullptr，所有权转移给 conn |
| lease 生命周期 | conn 析构时自动释放 `account::lease` |

## 失败场景

### 1. 拨号失败后客户端收到 502

上游服务器不可达或 DNS 解析失败。`connect::dial` 返回错误码或空传输层。`handle()` 调用 `relay->send_gateway_err()` 发送 502 响应后返回。客户端可以据此重试或报告错误。

### 2. 正向代理链路断开

配置了 `outbound_proxy` 时，正向代理服务器不可达。拨号失败路径与直接拨号相同，客户端收到 502。区别在于日志中会显示正向代理层面的错误信息。

### 3. CONNECT 响应写入失败

`send_ok()` 写入 `200 Connection Established` 失败（客户端已断开）。`handle()` 直接返回，不进入隧道。上游连接（如有）通过 `outbound` shared_ptr 引用计数归零后自动关闭。

## 跨模块契约

| 上游模块 | 契约 |
|----------|------|
| [[core/protocol/http/conn]] | 提供 `make_conn`、`handshake`、`send_ok`、`send_gateway_err`、`forward`、`release` |
| [[core/protocol/http/parser]] | 提供 `proxy_request` 结构和 `parse_req` |
| [[core/transport/preview]] | 提供 `wrap_with_preview` 装饰器 |
| [[core/connect/dial/dial]] | 提供拨号功能（直接路由和正向代理两种路径） |
| [[core/connect/tunnel/tunnel]] | 提供双向隧道转发 |
| [[core/recognition/target]] | 提供 `recognition::resolve`，从 proxy_request 解析目标地址 |
| [[core/context/context]] | 提供 `session` 上下文 |

## 变更敏感度

| 修改点 | 影响范围 |
|--------|----------|
| 方法分发逻辑 | CONNECT/非 CONNECT 的行为差异，调换顺序会导致隧道建立失败 |
| 拨号路径选择 | `outbound_proxy` 的判断条件，影响所有 HTTP 代理连接的路由 |
| `recognition::resolve` 接口 | 目标地址解析逻辑变更会影响所有 HTTP 请求的路由 |
| `forward` 的 URI 重写 | 绝对 URI → 相对路径的转换规则，修改会导致源站收到错误请求 |

## 相关文档

- [[core/protocol/http/conn]] — HTTP 代理中继器
- [[core/protocol/http/parser]] — 请求解析、认证、URI 重写
- [[core/protocol/http/relay]] — 旧版 relay 文档
- [[core/transport/preview]] — 预读重放装饰器
- [[core/connect/dial/dial]] — 拨号连接
- [[core/connect/tunnel/tunnel]] — 双向隧道转发
- [[core/recognition/target]] — 目标地址解析
