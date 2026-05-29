---
layer: core
source: include/prism/protocol/vless/process.hpp
  - "include/prism/protocol/vless/process.hpp"
  - "src/prism/protocol/vless/process.cpp"
  - "include/prism/protocol/vless/packet.hpp"
title: VLESS 处理入口与请求结构
tags: [protocol, vless, process, request, mux, udp, tcp]
---

# VLESS 处理入口与请求结构

> 源码位置: `include/prism/protocol/vless/process.hpp`
> 实现位置: `src/prism/protocol/vless/process.cpp`

## 模块概述

VLESS 处理入口（`handle` 函数）是 VLESS 协议的业务逻辑入口，协调完整的请求处理流程：传输层包装 → 中继器握手 → 命令分发 → 数据转发。

该函数由 Protocol Layer 的分发表（dispatch table）在协议识别为 VLESS 后调用，是连接 Recognition Layer 和 Protocol Layer 的桥梁。

请求结构（`request`）定义了 VLESS 协议握手成功后返回的解析结果，包含 UUID、命令、目标地址等完整信息。

## 命名空间

```cpp
namespace psm::protocol::vless
```

## 请求结构

### request

```cpp
struct request
{
    std::array<std::uint8_t, 16> uuid;                            // 用户 UUID（16 字节原始数据）
    command cmd;                                                  // 命令类型
    std::uint16_t port;                                           // 目标端口，主机字节序
    address destination_address;                                  // 目标地址（variant: ipv4/ipv6/domain）
    psm::protocol::form transport = psm::protocol::form::stream;  // 传输形式，由命令类型决定
};
```

**字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `uuid` | `array<uint8_t, 16>` | 用户凭证，二进制格式，由 `conn::handshake()` 从请求头 memcpy 填充 |
| `cmd` | `command` | 命令类型：`tcp`(0x01)、`udp`(0x02)、`mux`(0x7F) |
| `port` | `uint16_t` | 目标端口，从大端字节序转换为主机字节序 |
| `destination_address` | `address` | 目标地址变体，引用 `protocol::common` 中的共享类型 |
| `transport` | `form` | 传输形态，`tcp`/`mux` 对应 `stream`，`udp` 对应 `datagram` |

**设计特点**:
- 地址类型通过 `using` 声明引用 `protocol::common` 中的共享定义（`ipv4_address`、`ipv6_address`、`domain_address`），避免跨协议重复
- 所有结构设计为零拷贝友好，可直接从协议缓冲区 `memcpy` 填充
- `transport` 字段由 `handshake()` 根据命令类型自动设置

### to_string

```cpp
[[nodiscard]] inline auto to_string(const address &addr,
                                     memory::resource_pointer mr = memory::current_resource())
    -> memory::string;
```

将地址变体转为可读字符串，委托给 `protocol::common::addr_to_str`。IPv4/IPv6 使用 `inet_ntop` 格式化，域名直接返回原始内容。支持 PMR 内存资源，适用于日志输出。

## handle 函数

```cpp
auto handle(context::session &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

VLESS 协议完整处理流程的入口函数。

**参数**:
- `ctx`: 会话上下文，包含服务器配置、工作线程资源、账户目录、传输层等
- `data`: 协议识别阶段的预读数据（通常 24 字节），需要回放给 VLESS 解析器

### 处理流程

```
           ┌─────────────────────────────────────────────────────────┐
           │  handle(ctx, data)                                      │
           │                                                          │
           │  1. wrap_with_preview(inbound, data)                    │
           │     └─ 将预读数据包装为 preview 回放层                   │
           │                                                          │
           │  2. 创建 verifier 回调                                   │
           │     └─ 绑定 account::try_acquire 查询                   │
           │                                                          │
           │  3. make_conn() → handshake()                           │
           │     └─ UUID 认证 + 请求解析                              │
           │                                                          │
           │  4. 根据 cmd 分发                                        │
           ├───────┬──────────────────┬───────────────────────────────┤
           │  tcp  │      udp         │          mux                  │
           │  ↓    │      ↓           │          ↓                    │
           │forward│ async_associate  │  multiplex::bootstrap         │
           │ tunnel│  UDP over TLS    │  sing-mux 协商                │
           └───────┴──────────────────┴───────────────────────────────┘
```

### 阶段详解

#### 阶段 1: 传输层包装

```cpp
auto inbound = psm::transport::wrap_with_preview(std::move(ctx.inbound), data, ctx.frame_arena.get());
ctx.inbound = nullptr;
```

协议识别阶段已从入站流预读了部分数据（`data`），需要通过 `preview` 包装器将其回放到数据流前端，确保 VLESS 握手能读取完整的请求头。

`ctx.inbound` 被置为 `nullptr`，所有权转移到 `inbound` 局部变量。

#### 阶段 2: 创建 verifier 回调

```cpp
auto verifier = [&ctx](const std::string_view credential) -> bool
{
    if (!ctx.account_directory) { return false; }
    auto lease = account::try_acquire(*ctx.account_directory, credential);
    if (!lease) { return false; }
    ctx.account_lease = std::move(lease);
    return true;
};
```

通过 lambda 捕获 `ctx` 引用，将 `conn` 的认证回调绑定到 `account::directory` 查询。关键行为：

1. 检查 `account_directory` 是否配置，未配置时拒绝所有连接
2. 调用 `account::try_acquire` 查询 UUID 凭证并获取连接租约
3. 租约（`account::lease`）以 RAII 方式管理，析构时自动递减活跃连接计数

#### 阶段 3: 握手

```cpp
const auto agent = make_conn(std::move(inbound), ctx.server_ctx.config().protocol.vless, std::move(verifier));
agent->set_traffic(ctx.worker_ctx.traffic, ctx.detected_protocol);
auto [vless_ec, req] = co_await agent->handshake();
```

创建 `conn` 实例，注入 VLESS 配置和验证回调，设置流量统计指针，然后执行握手。握手失败时记录警告日志并返回。

#### 阶段 4a: TCP / MUX 命令

```cpp
case command::tcp:
case command::mux:
```

TCP 和 MUX 命令共享同一处理路径，区别在于 MUX 路径会检测多路复用标记地址。

**目标地址解析**: 将 `request` 中的地址变体转为 `target` 结构，端口通过 `std::to_chars` 转为十进制字符串。

**多路复用检测**:
```cpp
auto mux_sw = ctx.server_ctx.config().mux.enabled
    ? psm::connect::mux_switch::on
    : psm::connect::mux_switch::off;
if (psm::connect::is_mux(target.host, mux_sw))
```

Mihomo / sing-box 兼容：客户端使用 mux 命令或以 `.mux.sing-box.arpa` 结尾的虚假地址标记多路复用连接。检测到后：

1. 清空 `stream_close` / `stream_cancel` 回调（多路复用自行管理生命周期）
2. 调用 `multiplex::bootstrap()` 进行 sing-mux 协商
3. 协商成功后启动 mux 会话

**TCP 直连路径**:
```cpp
co_await psm::connect::forward(ctx, {"Vless", target, agent->release()});
```

`agent->release()` 将底层传输层所有权转移给 `connect::forward`，后者执行 dial（拨号连接上游）+ tunnel（双向转发）。

#### 阶段 4b: UDP 命令

```cpp
case command::udp:
```

创建 UDP 路由回调，支持两种来源：

1. **出站代理模式**: `ctx.outbound_proxy->make_router()` — 通过上游代理转发 UDP
2. **直连模式**: `psm::connect::make_router(ctx.worker_ctx.router)` — 通过本地路由器解析

路由回调接收主机名和端口字符串，返回 `udp::endpoint`。

实际 UDP 帧循环由 `agent->async_associate()` 执行。

## 设计决策

### WHY: 预读数据通过 preview 回放

**Problem**: Recognition Layer 在协议识别阶段已从传输层读取了部分数据（通常 24 字节），这些数据属于 VLESS 请求头的一部分，后续解析器必须能读取到。

**Choice**: 使用 `transport::preview` 包装器，将预读数据保存在内部缓冲区中，后续读取时优先返回预读数据，耗尽后再委托底层传输。

**Consequence**: VLESS 解析器无需关心数据是否已被预读，接口一致性得到保证。`preview` 在构造时复制数据到 PMR 缓冲区，确保数据生命周期安全。

### WHY: verifier lambda 捕获 ctx 引用

**Problem**: `conn` 的认证回调需要在握手过程中查询账户目录并获取租约，但 `conn` 本身不应持有会话上下文的引用。

**Choice**: 在 `handle` 函数中创建 lambda，按引用捕获 `ctx`，将 `conn` 的通用回调绑定到具体的账户查询逻辑。

**Consequence**: 租约（`account::lease`）存储在 `ctx.account_lease` 中，随会话生命周期自动释放。如果 `handle` 在 `verifier` 调用前已退出（不会发生，因为握手是同步协程），引用会悬空——但协程保证了执行顺序。

### WHY: TCP 和 MUX 命令合并处理

**Problem**: MUX 命令在协议层面与 TCP 几乎相同，区别仅在于是否使用多路复用标记地址。

**Choice**: 在 `switch` 中将 `tcp` 和 `mux` 合并到同一 case，通过 `connect::is_mux()` 运行时检测多路复用标记。

**Consequence**: 代码路径简洁，避免重复的目标地址解析和日志记录。多路复用检测基于目标主机名后缀（`.mux.sing-box.arpa`），与 Mihomo / sing-box 兼容。

### WHY: 两种 UDP 路由来源

**Problem**: UDP 数据报的转发路径取决于部署模式——直连或通过上游代理。

**Choice**: 运行时检查 `ctx.outbound_proxy` 是否存在，存在时使用代理路由，否则使用本地路由器。

**Consequence**: 同一套 UDP 帧循环逻辑支持两种转发模式，无需在协议层区分部署模式。

## 约束

- **request 零拷贝**: `uuid` 字段直接从协议缓冲区 `memcpy`，不经过中间转换
- **MUX 检测依赖配置**: 多路复用开关 `mux_switch` 由服务端配置控制，客户端无法独立决定
- **UDP 路由回调生命周期**: `dgram_router` 作为局部变量传入 `async_associate`，回调通过 `std::function` 持有，必须保证在帧循环期间有效
- **预读数据大小**: 依赖 Recognition Layer 的 probe 预读量（通常 24 字节），必须小于 VLESS 最小请求长度（26 字节）才不会丢失有效数据

## 失败场景

| 场景 | 错误码 | 行为 |
|------|--------|------|
| 账户目录未配置 | — | `verifier` 返回 `false`，握手返回 `auth_failed` |
| UUID 凭证不存在或连接数已满 | — | `try_acquire` 返回空，`verifier` 返回 `false` |
| 握手失败 | `fault::code` 系列 | 记录警告日志，`co_return` 退出 |
| MUX 协商失败 | — | `bootstrap` 返回 `nullptr`，不启动会话 |
| UDP associate 失败 | `fault::code` 系列 | 记录警告日志 |
| 未知命令 | — | 记录警告日志，`default` 分支忽略 |

## 跨模块契约

### 上游依赖

| 模块 | 契约 |
|------|------|
| `context::session` | 会话上下文，提供所有运行时资源 |
| `transport::preview` | 预读数据回放，`wrap_with_preview` 工厂函数 |
| `vless::conn` | 连接中继器，执行握手和 UDP associate |
| `account::directory` | 账户查询，通过 `try_acquire` 获取租约 |
| `account::lease` | RAII 连接租约，析构时释放连接计数 |
| `connect::forward` | TCP 隧道转发（dial + tunnel） |
| `connect::util` | `is_mux()` 多路复用标记检测 |
| `multiplex::bootstrap` | 多路复用会话引导（sing-mux 协商） |
| `outbound::proxy` | 出站代理，提供 UDP 路由回调 |

### 下游消费者

| 模块 | 契约 |
|------|------|
| Protocol Layer 分发表 | 通过函数指针调用 `handle`，传入会话上下文和预读数据 |
| Recognition Layer | 识别 VLESS 协议后，将控制权转移给 `handle` |

## 变更敏感度

| 修改 | 影响范围 |
|------|----------|
| 新增命令类型 | `handle` 的 switch 分支、`conn::parse_header` 的命令校验 |
| 修改 MUX 检测逻辑 | `connect::is_mux`、MUX 标记地址格式 |
| 修改认证流程 | `verifier` lambda、`account::try_acquire` 签名 |
| 修改预读数据量 | probe 的预读大小、`wrap_with_preview` 的数据 span |
| 新增 UDP 路由来源 | `dgram_router` 创建逻辑 |
| 修改 request 结构 | 所有解析和消费 `request` 的代码路径 |

## 调用链

- [[core/protocol/vless/conn|Conn]] — 连接中继器，执行握手
- [[core/protocol/vless/packet|Packet]] — 请求结构定义（本文件）
- [[core/protocol/vless/constants|Constants]] — 命令枚举
- [[core/protocol/vless/config|Config]] — 协议配置
- [[core/protocol/vless/format|Format]] — 协议格式编解码
- [[core/protocol/common/address|Address]] — 共享地址类型
- [[core/transport/preview|Preview]] — 预读数据回放
- [[core/connect/forward|Forward]] — TCP 隧道转发
- [[core/connect/util|Connect Utils]] — MUX 检测工具
- [[core/multiplex/bootstrap|Multiplex Bootstrap]] — 多路复用引导
- [[core/account/entry|Account Entry]] — 连接租约管理
- [[core/context/context|Context]] — 三层上下文结构
