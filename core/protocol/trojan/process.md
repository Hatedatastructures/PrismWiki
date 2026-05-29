---
layer: core
source:
  - "include/prism/protocol/trojan/process.hpp"
  - "src/prism/protocol/trojan/process.cpp"
  - "include/prism/protocol/trojan/packet.hpp"
title: Trojan 协议处理入口与请求结构
tags: [protocol, trojan, process, request, dispatch, connect, udp, mux]
---

# Trojan 协议处理入口与请求结构

> 源码: `include/prism/protocol/trojan/process.hpp` | 实现: `src/prism/protocol/trojan/process.cpp`
> 数据结构: `include/prism/protocol/trojan/packet.hpp`

## 模块概述

Trojan 协议的完整处理流程入口函数 `handle()` 和请求消息结构 `request`。`handle()` 是 Trojan 协议在 session 层的唯一入口点，协调凭据验证、握手、命令分发和隧道转发的完整生命周期。

`request` 结构由握手过程填充，携带命令类型、目标地址、端口和凭据信息，是后续路由和转发决策的核心数据。

## handle 函数

```cpp
namespace psm::protocol::trojan {
    auto handle(context::session &ctx, std::span<const std::byte> data)
        -> net::awaitable<void>;
}
```

**参数**:

| 参数 | 说明 |
|------|------|
| `ctx` | 会话上下文，聚合服务器配置、工作线程资源、账户目录等 |
| `data` | 协议嗅探阶段的预读数据，通过 `preview` 包装器回放 |

### 处理流程

```
handle(ctx, data)
     │
     ▼
wrap_with_preview(inbound, data)      ← 预读数据回放
     │
     ▼
构建凭据验证器 (verifier lambda)      ← account_directory + try_acquire
     │
     ▼
make_conn(inbound, trojan_cfg, verifier)  ← 创建连接中继器
agent->set_traffic(...)                   ← 设置流量统计
     │
     ▼
agent->handshake()                     ← 协议握手
     │
     ├── 失败 → 日志 + 返回
     │
     ▼
根据 req.cmd 分发:
     │
     ├─ command::connect
     │     │
     │     ├─ 构造 target（host/port）
     │     │
     │     ├─ mux 检测 (is_mux)        ← ".mux.sing-box.arpa" 后缀
     │     │     └─ 命中 → multiplex::bootstrap → co_return
     │     │
     │     └─ connect::forward(ctx, {"Trojan", target, agent->release()})
     │
     ├─ command::udp_associate
     │     │
     │     ├─ 构建 dgram_router        ← outbound_proxy 或 connect::make_router
     │     └─ agent->async_associate(dgram_router)
     │
     ├─ command::mux (0x7F)
     │     └─ multiplex::bootstrap → co_return
     │
     └─ default → 日志警告
```

### 凭据验证器

`handle` 内部构建 lambda 作为凭据验证回调：

```cpp
auto verifier = [&ctx](const std::string_view credential) -> bool
{
    if (!ctx.account_directory) return false;
    auto lease = account::try_acquire(*ctx.account_directory, credential);
    if (!lease) return false;
    ctx.account_lease = std::move(lease);
    return true;
};
```

**行为**:

1. 检查 `account_directory` 是否已配置，未配置直接拒绝
2. 调用 `account::try_acquire()` 尝试获取连接租约
3. 成功时将租约存储到 `ctx.account_lease`，保持会话级连接计数

> **约束**: 验证器通过引用捕获 `ctx`，生命周期绑定到 `handle` 协程帧。`try_acquire` 可能执行连接数限制检查。

### MUX 兼容处理

Trojan 协议支持两种多路复用入口：

1. **Mihomo smux 兼容模式**: 客户端使用 `command::connect`（0x01）+ 虚假目标地址 `.mux.sing-box.arpa` 标记 mux 连接。通过 `connect::is_mux()` 检测。
2. **原生 mux 模式**: 客户端使用 `command::mux`（0x7F），直接进入多路复用模式。

两种模式最终都调用 `multiplex::bootstrap()` 创建多路复用会话，并清空 `ctx.stream_close` 和 `ctx.stream_cancel`（多路复用会话自行管理生命周期）。

### UDP 路由选择

UDP_ASSOCIATE 命令的路由回调来源：

- 若配置了 `outbound_proxy`：使用 `outbound_proxy->make_router()`
- 否则：使用 `connect::make_router(ctx.worker_ctx.router)`

## request 结构

```cpp
struct request
{
    command cmd;                                        // 命令类型
    std::uint16_t port;                                 // 目标端口，主机字节序
    address destination_address;                        // 目标地址变体
    std::array<char, 56> credential;                    // 用户凭据，56 字节 SHA224 hex
    psm::protocol::form transport = form::stream;       // 传输形式
};
```

**字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `cmd` | `command` | CONNECT(0x01)、UDP_ASSOCIATE(0x03)、MUX(0x7F) |
| `port` | `uint16_t` | 目标端口，解析后为主机字节序 |
| `destination_address` | `address` | `variant<ipv4_address, ipv6_address, domain_address>` |
| `credential` | `array<char, 56>` | 密码的 SHA224 hex 哈希 |
| `transport` | `form` | `stream`(TCP) 或 `datagram`(UDP)，由命令类型决定 |

**地址类型**: 使用 `using` 声明引用 `protocol::common` 中的共享类型，避免跨协议重复定义：

```cpp
using protocol::common::address;
using protocol::common::domain_address;
using protocol::common::ipv4_address;
using protocol::common::ipv6_address;
```

### 地址字符串化

```cpp
[[nodiscard]] inline auto to_string(const address &addr,
    memory::resource_pointer mr = memory::current_resource()) -> memory::string;
```

委托给 `protocol::common::addr_to_str()`，支持 PMR 内存分配。IPv4/IPv6 使用 `inet_ntop` 格式化，域名直接返回原始内容。

## 设计决策

### WHY: handle 作为协程入口而非类

**问题**: 协议处理器需要协调多个组件（中继器、路由器、隧道），且生命周期绑定到单个会话。

**选择**: 使用自由函数 `handle()` 返回 `net::awaitable<void>`，所有中间状态存储在协程帧和 `context::session` 中。

**后果**: 无额外堆分配（协程帧由编译器管理），但 `handle` 内的 lambda 和局部变量生命周期绑定到协程帧，需确保引用的 `ctx` 在协程完成前有效。

### WHY: 凭据验证器在 handle 内构建而非外部注入

**问题**: 凭据验证需要访问 `account_directory` 和连接限制，且成功后需存储 `account_lease`。

**选择**: 在 `handle()` 内部构建 lambda，通过引用捕获 `ctx`。

**后果**: 验证逻辑与进程入口紧耦合，但避免了额外的抽象层和回调传递。验证器生命周期与 `handle` 协程帧一致。

### WHY: 两种 MUX 入口路径

**问题**: 不同客户端实现（Mihomo vs 原生 Trojan）使用不同的命令字发起多路复用。

**选择**: 同时支持 `command::connect` + `.mux.sing-box.arpa` 后缀检测和 `command::mux`(0x7F) 两种入口。

**后果**: 两种路径最终调用相同的 `multiplex::bootstrap()`，但入口检测逻辑增加了一层条件分支。`is_mux` 检测依赖配置中 `mux.enabled` 开关。

## 约束

| 约束 | 说明 |
|------|------|
| **会话单入口** | 每个会话只调用一次 `handle()`，完成后会话结束 |
| **预读数据必须回放** | `data` 中的预读字节通过 `preview` 包装器重放，不可跳过 |
| **account_directory 可选** | 未配置时所有凭据验证失败 |
| **Mux 需全局启用** | Mihomo 兼容模式依赖 `config().mux.enabled` |
| **传输层所有权转移** | `agent->release()` 后 `conn` 不再持有传输层 |

## 故障场景

| 场景 | 行为 |
|------|------|
| account_directory 未配置 | 验证器返回 false，握手失败 |
| 凭据验证失败 | `auth_failed`，`handle` 记录日志后返回 |
| 握手超时 | 30 秒超时，`handle` 返回 |
| MUX bootstrap 返回空 | `muxprotocol` 为空，不启动，静默返回 |
| 拨号失败 | `connect::forward` 内部处理错误，隧道终止 |
| UDP 路由失败 | `async_associate` 内部处理，返回错误码 |
| outbound_proxy 为空 | UDP 使用默认 `connect::make_router` |

## 跨模块契约

### 上游依赖（handle 使用）

| 模块 | 契约 |
|------|------|
| [[core/protocol/trojan/conn]] | `make_conn` 创建中继器，`handshake()` 执行握手，`release()` 转移传输层 |
| [[core/protocol/trojan/config]] | 通过 `ctx.server_ctx.config().protocol.trojan` 获取配置 |
| [[core/protocol/trojan/packet]] | `request` 结构作为握手结果 |
| [[core/transport/preview]] | `wrap_with_preview` 包装预读数据回放 |
| [[core/connect/tunnel/forward]] | TCP 隧道转发（`connect::forward`） |
| [[core/connect/dial/dial]] | `make_router` 创建 UDP 路由回调 |
| [[core/connect/util]] | `is_mux` 检测 mux 标记地址 |
| [[core/multiplex/bootstrap]] | 多路复用会话引导 |
| [[core/account/directory]] | `try_acquire` 凭据验证和连接租约 |
| [[core/context/context]] | `session` 上下文提供所有运行时资源 |

### 下游消费者（调用 handle）

| 模块 | 契约 |
|------|------|
| session 层 (`instance/session`) | 会话识别出 Trojan 协议后调用 `trojan::handle()` |

## 变更敏感性

| 修改点 | 影响范围 |
|--------|----------|
| `handle` 函数签名 | session 层调用点 |
| `request` 结构字段 | `conn::handshake()` 填充逻辑和 `handle` 中所有使用处 |
| `command` 枚举值 | `handle` 中的 `switch` 分支和 `validate_command` |
| account 接口变更 | 凭据验证器 lambda |
| `forward_options` 结构 | CONNECT 路径的参数传递 |
| mux 配置路径 | Mihomo 兼容模式的 `is_mux` 检测条件 |
