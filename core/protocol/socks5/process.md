---
title: SOCKS5 协议处理入口
layer: core
module: protocol/socks5
source:
  - include/prism/protocol/socks5/process.hpp
  - src/prism/protocol/socks5/process.cpp
  - include/prism/protocol/socks5/packet.hpp
tags: [protocol, socks5, process]
---

# SOCKS5 协议处理入口

`socks5::handle()` 是 SOCKS5 协议的顶层入口函数，协调完整的请求处理流程：预读回放 → 握手 → 命令分发 → 拨号 → 隧道转发。它是 `session::diversion()` 通过 Dispatch Layer 路由到 SOCKS5 处理器的第一个调用点。

## 模块组成

| 组件 | 功能 | 文件 |
|------|------|------|
| `handle()` | 协议入口函数，编排完整流程 | process.hpp / process.cpp |
| `request` | SOCKS5 请求结构体，封装命令/地址/端口/传输形态 | packet.hpp |
| `address` 类型别名 | 引用 `protocol::common` 共享地址类型 | packet.hpp |
| `to_string()` | 地址变体转字符串，用于日志和路由 | packet.hpp |

## 核心函数

### handle

```cpp
auto handle(context::session &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

SOCKS5 协议的完整处理入口，由 Dispatch Layer 在识别到 SOCKS5 协议后调用。

**参数**：
- `ctx`：会话上下文，包含入站传输、worker 资源、出站代理等
- `data`：Recognition 阶段的预读数据（已被消耗的字节，需回放）

**处理流程**：

```
wrap_with_preview(ctx.inbound, data)
        ↓                        // 将预读数据包装为可回放传输层
make_conn(inbound, config, account_dir)
        ↓                        // 创建 SOCKS5 中继器
agent->set_traffic(traffic, proto)
        ↓                        // 设置流量统计
agent->handshake()
        ↓                        // 执行 SOCKS5 握手
   ┌─ command::connect
   │     ↓
   │  to_string(addr) → target   // 构建拨号目标
   │     ↓
   │  connect::dial(proxy/router, target)  // 拨号上游
   │     ↓ (成功)
   │  agent->send_success(req)   // 发送 SOCKS5 成功响应
   │     ↓
   │  connect::tunnel({release(), outbound, ctx})  // 双向隧道转发
   │
   ├─ command::udp_associate
   │     ↓
   │  make_router(proxy/router)  // 创建 UDP 数据报路由
   │     ↓
   │  agent->async_associate(req, router)  // 进入 UDP 转发模式
   │
   └─ default (BIND 等)
          ↓
       agent->send_error(cmd_unsupported)
```

### CONNECT 分支

TCP 连接请求的处理细节：

1. **目标构建**：将 `request.destination_address` 转为字符串、`destination_port` 转为十进制字符串，填充 `target` 结构体（`positive=true`）
2. **拨号路由**：优先使用 `ctx.outbound_proxy` 拨号（正向代理链），否则使用 `ctx.worker_ctx.router` 路由
3. **错误映射**：拨号失败时将内部错误码映射为 RFC 1928 语义的 `reply_code`
   - `fault::code::ipv6_disabled` → `reply_code::network_unreachable`
   - 其他拨号错误 → `reply_code::host_unreachable`
4. **隧道建立**：拨号成功后发送 SOCKS5 成功响应，再通过 `agent->release()` 转移传输层给 `connect::tunnel()`

### UDP_ASSOCIATE 分支

UDP 关联请求的处理细节：

1. **路由器创建**：根据是否配置出站代理，选择 `outbound_proxy->make_router()` 或 `connect::make_router(router)`
2. **关联处理**：调用 `agent->async_associate(request, router)` 进入 UDP 数据报转发循环
3. **生命周期**：UDP 关联由 `conn` 内部管理，退出时自动刷写流量统计

## 数据结构

### request

```cpp
struct request
{
    command cmd;                        // 命令类型
    std::uint16_t destination_port;     // 目标端口（主机字节序）
    address destination_address;        // 目标地址（IPv4/IPv6/域名变体）
    form transport = form::stream;      // 传输形态（stream 或 datagram）
};
```

由 `conn::handshake()` 解析填充，`handle()` 消费使用。关键字段含义：

| 字段 | 来源 | 消费方 |
|------|------|--------|
| `cmd` | `wire::header_parse.cmd` | `handle()` 的 switch 分发 |
| `destination_port` | `wire::decode_port()` | `process.cpp` 转为字符串传给 `dial()` |
| `destination_address` | `wire::parse_ipv4/ipv6/domain()` | `to_string()` 转为字符串 |
| `transport` | `conn::resolve_command()` 根据 cmd 设置 | `async_associate()` 验证 form 类型 |

### address 类型别名

`packet.hpp` 通过 `using` 声明引用 `protocol::common` 中的共享地址类型：

```cpp
using protocol::common::address;          // std::variant<ipv4_address, ipv6_address, domain_address>
using protocol::common::domain_address;   // 域名地址
using protocol::common::ipv4_address;     // IPv4 地址（4 字节数组）
using protocol::common::ipv6_address;     // IPv6 地址（16 字节数组）
```

零拷贝友好，跨协议复用，避免 SOCKS5/Trojan/VLESS 重复定义地址类型。

## 设计决策

### 为什么 handle() 接受预读数据而非让 conn 自行读取？

**问题**: Recognition 流水线在协议识别阶段已经从传输层读取了部分数据（前 24 字节用于 Probe 检测），这些数据属于 SOCKS5 协议报文的一部分。

**选择**: `handle()` 接受 `std::span<const std::byte> data` 参数，通过 `transport::wrap_with_preview()` 将预读数据注入到传输层前端。`conn` 在后续读取时先消耗预读数据，耗尽后再从底层传输层读取。

**后果**: `conn` 无需感知预读机制，保持了协议处理的纯粹性。但 `wrap_with_preview()` 需要将预读数据复制到内部缓冲区，有一次小的内存拷贝开销。

### 为什么 CONNECT 先拨号后发送成功响应？

**问题**: SOCKS5 协议允许先发送成功响应再建立连接，也可以反过来。

**选择**: 先拨号建立上游连接，成功后再发送 SOCKS5 成功响应。拨号失败时发送对应的错误响应码。

**后果**: 客户端在收到成功响应时上游连接已就绪，可以立即开始数据传输，减少端到端延迟。但拨号耗时较长的场景下，客户端可能因等待 SOCKS5 响应而超时。错误映射确保客户端能区分"网络不可达"和"主机不可达"等不同失败原因。

### 为什么 UDP_ASSOCIATE 使用 `outbound::router_fn` 而非 `connect::dial`？

**问题**: UDP 数据报是逐包路由的，每个数据报可能发往不同目标，不适合像 TCP 那样建立持久隧道。

**选择**: 使用 `outbound::router_fn`（类型为 `function<awaitable<pair<fault::code, udp::endpoint>>(string_view host, string_view port)>`），每次收到 UDP 数据报时调用路由回调查询目标端点。

**后果**: UDP 路由决策由 `outbound::proxy` 或 `connect::router` 处理，`process` 层不关心路由细节。出站代理链对 TCP 和 UDP 使用统一的 `router_fn` 接口。

### 为什么 `request` 使用 `form` 而非 `command` 区分传输形态？

**问题**: SOCKS5 命令（CONNECT/UDP_ASSOCIATE/BIND）和传输形态（TCP/UDP）是两个维度，但命令唯一确定了传输形态。

**选择**: `conn::resolve_command()` 根据 `command` 设置 `request.transport` 字段（CONNECT → `form::stream`，UDP_ASSOCIATE → `form::datagram`）。`handle()` 和下游模块通过 `form` 判断传输形态。

**后果**: 下游模块无需了解 SOCKS5 命令语义，只关注传输形态。Trojan 和 VLESS 协议也使用相同的 `form` 枚举，实现了跨协议的传输形态统一。

## 约束

### 会话上下文完整性

**类型**: 前置条件

**规则**: `handle()` 要求 `ctx.inbound` 非空、`ctx.server_ctx.config()` 包含有效的 `socks5` 配置段、`ctx.worker_ctx.io_context` 有效

**违反后果**: 空指针解引用或配置访问异常

**源码依据**: process.cpp:22-32

### 预读数据范围

**类型**: 数据约束

**规则**: `data` 参数的长度必须与 Recognition 阶段实际预读的字节数一致

**违反后果**: 数据丢失或重复，导致 SOCKS5 握手解析失败

**源码依据**: process.cpp:22

### 出站代理与路由器的互斥

**类型**: 配置约束

**规则**: `ctx.outbound_proxy` 非空时使用代理拨号，为空时使用 `ctx.worker_ctx.router`。两者不可同时为活跃状态

**违反后果**: 拨号路径选择错误，连接目标不正确

**源码依据**: process.cpp:59-67

## 故障场景

### 入站传输层缺失

**触发条件**: `ctx.inbound` 在调用 `handle()` 前已被移动或释放

**传播路径**: `wrap_with_preview()` 返回空 → `trace::warn` → `co_return`

**外部表现**: 连接静默断开，无 SOCKS5 响应

**恢复机制**: 确保调用链正确传递 `inbound` 所有权

**日志关键字**: `"[Protocol.Socks5] inbound missing"`

### 握手失败

**触发条件**: 客户端发送非法 SOCKS5 数据、版本不匹配、认证失败、超时

**传播路径**: `agent->handshake()` 返回非 success 错误码 → `trace::error` → `co_return`

**外部表现**: 连接断开，客户端收到错误响应或连接重置

**恢复机制**: 客户端重新建立连接

**日志关键字**: `"[Protocol.Socks5] handshake failed"`

### 上游拨号失败

**触发条件**: 目标主机不可达、DNS 解析失败、连接超时、IPv6 禁用

**传播路径**: `connect::dial()` 返回错误 → `agent->send_error(reply_code)` → `co_return`

**外部表现**: 客户端收到 RFC 1928 语义的错误响应

| 错误码 | SOCKS5 响应 | 含义 |
|--------|-------------|------|
| `ipv6_disabled` | `network_unreachable` (0x03) | IPv6 被策略禁用 |
| 其他 | `host_unreachable` (0x04) | 目标主机不可达 |

**恢复机制**: 客户端根据 SOCKS5 响应码决策重试策略

**日志关键字**: `"[Protocol.Socks5] dial failed"`、`"[Protocol.Socks5] IPv6 disabled"`

### 成功响应发送失败

**触发条件**: 拨号成功后，向客户端发送 SOCKS5 成功响应时连接已断开

**传播路径**: `agent->send_success()` 返回错误 → `co_return`（不关闭上游连接，由 shared_ptr 析构释放）

**外部表现**: 上游连接被浪费（已建立但未使用），客户端无响应

**恢复机制**: 无（上游连接由 shared_ptr 引用计数自动释放）

### UDP_ASSOCIATE 失败

**触发条件**: UDP 端口绑定失败、路由器不可用、内部错误

**传播路径**: `agent->async_associate()` 返回错误 → `trace::warn` → `co_return`

**外部表现**: UDP 关联终止

**恢复机制**: 客户端重新发起 UDP_ASSOCIATE 请求

**日志关键字**: `"[Protocol.Socks5] UDP_ASSOCIATE failed"`

## 跨模块契约

| 模块 A | 模块 B | 契约内容 |
|--------|--------|---------|
| `process` | [[core/protocol/socks5/conn\|conn]] | `process` 创建 `conn` 并调用 `handshake()`，依赖返回的 `request` 包含有效的 `cmd`、`destination_address`、`destination_port` |
| `process` | [[core/transport/preview\|preview]] | `process` 通过 `wrap_with_preview()` 将预读数据注入传输层，`preview` 必须先返回预读字节再委托到底层 |
| `process` | [[core/connect/dial\|dial]] | `process` 调用 `connect::dial()` 建立 TCP 上游，拨号成功返回 `shared_transmission` |
| `process` | [[core/connect/tunnel\|tunnel]] | `process` 将 `agent->release()` 的传输层和 `outbound` 传给 `connect::tunnel()`，要求两者均有效 |
| `process` | [[core/outbound/proxy\|proxy]] | `process` 检查 `ctx.outbound_proxy` 非空时通过代理拨号，否则直连路由 |
| `process` | [[core/connect/dial\|connect::make_router]] | UDP 分支调用 `connect::make_router(router)` 创建数据报路由器 |
| `packet` | [[core/protocol/common/address\|address]] | `packet.hpp` 使用 `using` 引用 `protocol::common` 地址类型，必须保持类型兼容 |
| `packet` | [[core/protocol/common/form\|form]] | `request.transport` 使用 `protocol::form` 枚举，区分 TCP 流和 UDP 数据报 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| 修改 `handle()` 签名 | Dispatch Layer 注册表、session 路由 | 编译失败 |
| 修改 `request` 结构体 | `conn::handshake()`、`process.cpp`、所有下游消费者 | 编译失败 |
| 修改 `command` 枚举值 | `handle()` 的 switch、`conn::resolve_command()` | 未知命令走 default 分支 |
| 修改 `reply_code` 枚举 | `send_error()` 调用、客户端行为 | 客户端收到非预期响应码 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| `conn::handshake()` 返回结构变化 | `process.cpp` 解构绑定 | `[ec, request]` 适配 |
| `conn::release()` 语义变化 | `process.cpp` 隧道建立 | 传输层所有权转移 |
| `connect::dial()` 重载变化 | `process.cpp` 两条拨号路径 | 代理/直连路径参数 |
| `wrap_with_preview()` 接口变化 | `process.cpp` 入站构造 | 预读数据注入 |
| `outbound::proxy::make_router()` 签名变化 | `process.cpp` UDP 路由器创建 | `router_fn` 类型兼容性 |
| `context::session` 字段变化 | `process.cpp` 全部引用 | `ctx` 字段访问 |

## 相关文档

- [[core/protocol/socks5/conn|conn]] - SOCKS5 协议中继器
- [[core/protocol/socks5/constants|constants]] - 协议常量定义
- [[core/protocol/socks5/config|config]] - 协议配置结构
- [[core/protocol/socks5/wire|wire]] - 线级编解码
- [[core/protocol/common/address|address]] - 共享地址类型
- [[core/protocol/common/form|form]] - 传输形态枚举
- [[core/transport/preview|preview]] - 预读数据回放
- [[core/connect/dial|dial]] - 连接拨号
- [[core/connect/tunnel|tunnel]] - 双向隧道转发
- [[core/outbound/proxy|proxy]] - 出站代理抽象
