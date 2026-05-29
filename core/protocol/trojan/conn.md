---
layer: core
source:
  - "include/prism/protocol/trojan/conn.hpp"
  - "src/prism/protocol/trojan/conn.cpp"
title: Trojan 协议连接中继器
tags: [protocol, trojan, conn, handshake, credential, command, dispatch, udp]
---

# Trojan 协议连接中继器

> 源码: `include/prism/protocol/trojan/conn.hpp` | 实现: `src/prism/protocol/trojan/conn.cpp`

## 模块概述

Trojan 协议连接中继器（`conn`），是传输层装饰器，负责协议握手、凭据验证和命令分发。继承自 `transport::transmission`，持有底层传输层的独占所有权，通过装饰器模式透明地增强底层传输层功能。

Trojan 协议基于 TLS，在应用层添加固定格式头部实现认证和流量伪装。`conn` 类本身不提供加密，依赖底层传输层（通常为 TLS）提供机密性。

核心职责：

1. **凭据验证** — 读取 56 字节 SHA224 哈希并调用外部验证器
2. **协议头部解析** — 解析命令（CMD）、地址类型（ATYP）、目标地址和端口
3. **命令分发** — 支持 CONNECT、UDP_ASSOCIATE、MUX 三种命令
4. **UDP 关联** — 通过 `async_associate` 进入 UDP over TLS 帧循环

## 类定义

```cpp
class conn : public psm::transport::transmission,
             public std::enable_shared_from_this<conn>
```

**继承关系**:

- `transport::transmission` — 统一传输层接口，支持链式装饰
- `std::enable_shared_from_this<conn>` — 协程中安全的 `shared_ptr` 管理

## 构造函数

```cpp
explicit conn(shared_transmission next_layer, const config &cfg = {},
              std::function<bool(std::string_view)> credential_verifier = nullptr);
```

| 参数 | 说明 |
|------|------|
| `next_layer` | 底层传输层，必须已建立连接，所有权转移 |
| `cfg` | 协议配置（TCP/UDP 开关、超时等） |
| `credential_verifier` | 凭据验证回调，为空时任何凭据均通过 |

> **约束**: `credential_verifier` 为 `nullptr` 时，握手过程中任何凭据都会通过。必须在外层确保传入了有效的验证器。

## 核心函数

### handshake

执行完整的 Trojan 协议握手，返回 `request` 结构或错误码。

```cpp
auto handshake() const -> net::awaitable<std::pair<fault::code, request>>;
```

**状态机流程**:

```
read_min(68B)           ─┬─ 失败 → timeout/bad_message
                        │
verify_credential(0-57) ─┬─ 凭据非法 → protocol_error
                        ├─ 验证失败 → auth_failed
                        └─ CRLF 非法 → protocol_error

parse_cmd_atyp(58-59)   ─┬─ 失败 → bad_message

补读（按地址类型）      ─┬─ 失败 → timeout/bad_message

parse_request_target    ─┬─ 地址/端口/CRLF 非法 → various

validate_command        ─┬─ 命令被禁 → forbidden
                        └─ 未知命令 → unsupported_command
```

**两阶段读取策略**:

1. 首次批量读取至少 68 字节（最小请求长度：56 凭据 + 2 CRLF + 1 CMD + 1 ATYP + 4 IPv4 + 2 PORT + 2 CRLF）
2. 根据地址类型计算完整请求长度，若数据不足则补读剩余字节

**超时保护**: 30 秒 `steady_timer`，超时后调用 `next_layer_->cancel()` 终止读取。被取消的读取返回 `fault::code::timeout` 而非 `canceled`。

### async_associate

处理 UDP_ASSOCIATE 命令，进入 UDP over TLS 帧循环。

```cpp
using route_callback = std::function<net::awaitable<
    std::pair<fault::code, net::ip::udp::endpoint>>
    (std::string_view, std::string_view)>;

auto async_associate(route_callback route_cb) const -> net::awaitable<fault::code>;
```

委托给 `protocol::common::frame_loop` 模板函数，使用 `format::parse_udp_pkt` 和 `format::build_udp_pkt` 作为编解码器。支持空闲超时（默认 60 秒）和流量统计。

### 透传方法

以下方法直接委托给底层传输层，`conn` 不做额外处理：

| 方法 | 说明 |
|------|------|
| `async_read_some` | 异步读取 |
| `async_write_some` | 异步写入 |
| `close` | 关闭传输层 |
| `cancel` | 取消待处理异步操作 |
| `executor` | 获取执行器 |

### 所有权管理

| 方法 | 说明 |
|------|------|
| `next_layer()` | 获取裸指针，不转移所有权 |
| `underlying()` | 获取底层传输层引用 |
| `release()` | 释放所有权，返回 `shared_transmission` |

`release()` 用于握手完成后将传输层转移给其他组件（如 `connect::forward` 或 `multiplex::bootstrap`）。

## 内部成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `next_layer_` | `shared_transmission` | 底层传输层，构造时转移所有权 |
| `config_` | `config` | 协议配置，运行期间不可修改 |
| `verifier_` | `std::function<bool(string_view)>` | 凭据验证回调 |
| `traffic_` | `traffic_state *` | 流量统计指针（可选） |
| `proto_` | `protocol_type` | 协议类型，用于流量分类 |

## 工厂函数

```cpp
using shared_conn = std::shared_ptr<conn>;

[[nodiscard]] inline auto make_conn(
    shared_transmission next_layer, const config &cfg = {},
    std::function<bool(std::string_view)> credential_verifier = nullptr
) -> shared_conn;
```

## 设计决策

### WHY: 装饰器模式传输层所有权

**问题**: Trojan 协议握手需要读取和解析协议头部，但握手完成后数据读写应直接透传到底层，避免额外开销。

**选择**: `conn` 持有 `next_layer_` 的 `shared_ptr` 独占所有权。握手完成后通过 `release()` 将所有权转移给下游组件（如 `connect::forward`）。

**后果**: 传输层在 `conn` 和下游组件之间零拷贝转移，无引用计数泄漏风险。但 `release()` 后调用任何 `conn` 方法是未定义行为。

### WHY: 两阶段读取而非一次性读取最大长度

**问题**: 域名地址最大 255 字节，完整请求最大约 320 字节。但大多数请求使用 IPv4（仅 68 字节）。

**选择**: 先读最小 68 字节，解析地址类型后再按需补读。

**后果**: IPv4 请求只需一次系统调用，域名请求需要两次。固定缓冲区 320 字节恰好覆盖最大请求（含 255 字节域名），无余量。

### WHY: 30 秒握手超时

**问题**: 恶意客户端可能建立连接后不发送数据，占用服务器资源。

**选择**: 使用 `steady_timer` 设置 30 秒超时，超时后 cancel 底层传输。

**后果**: 正常客户端在 RTT 内完成握手不受影响。超时时间硬编码，不可配置。若需调整需修改源码。

## 约束

| 约束 | 说明 |
|------|------|
| **无加密** | `conn` 不提供加密，必须与 TLS 传输层组合使用 |
| **非线程安全** | 单实例应在同一协程或 strand 内使用 |
| **配置不可变** | `config_` 在构造时传入，运行期间不可修改 |
| **凭据明文** | 56 字节 SHA224 hex 明文在 TLS 内层传输 |

## 故障场景

| 场景 | 错误码 | 处理 |
|------|--------|------|
| 客户端不发送数据 / 网络中断 | `timeout` | 30 秒超时后 cancel 连接 |
| 凭据格式非法（非 hex 字符） | `protocol_error` | 关闭连接 |
| 凭据验证失败 | `auth_failed` | 关闭连接 |
| CRLF 分隔符缺失 | `protocol_error` | 关闭连接 |
| CONNECT 被禁用 | `forbidden` | 配置 `enable_tcp=false` |
| UDP_ASSOCIATE 被禁用 | `forbidden` | 配置 `enable_udp=false` |
| 未知命令字 | `unsupported_command` | 关闭连接 |
| 未知地址类型 | `unsupported_address` | 关闭连接 |
| 消息长度不足 | `bad_message` | 关闭连接 |
| 连接在读取中关闭 | `eof` | 正常关闭 |

## 跨模块契约

### 上游依赖（conn 使用）

| 模块 | 契约 |
|------|------|
| [[core/protocol/trojan/format]] | 调用 `parse_credential`、`parse_crlf`、`parse_cmd_atyp` 解析协议头部 |
| [[core/protocol/trojan/config]] | 读取 `enable_tcp`、`enable_udp`、`idle_timeout`、`max_dgram` |
| [[core/protocol/trojan/constants]] | 使用 `command` 和 `address_type` 枚举 |
| [[core/protocol/trojan/packet]] | 填充 `request` 结构并返回 |
| [[core/protocol/common/read]] | 调用 `read_min` 和 `read_remaining` 进行批量读取 |
| [[core/protocol/common/udprelay]] | 委托 `frame_loop` 处理 UDP over TLS |
| [[core/transport/preview]] | 入站传输层预读数据回放（由 process 层包装） |
| [[core/fault/code]] | 所有错误码通过 `fault::code` 返回 |

### 下游消费者（使用 conn）

| 模块 | 契约 |
|------|------|
| [[core/protocol/trojan/process]] | 调用 `make_conn` 创建实例，执行 `handshake()`，根据结果分发命令 |
| [[core/connect/tunnel/forward]] | 通过 `release()` 获取传输层进行 TCP 隧道转发 |
| [[core/multiplex/bootstrap]] | 通过 `release()` 获取传输层进入多路复用模式 |

## 变更敏感性

| 修改点 | 影响范围 |
|--------|----------|
| `handshake()` 返回类型 | `process.cpp` 中所有命令分发逻辑 |
| `request` 结构字段 | `process.cpp` 中地址解析和路由 |
| `config` 字段名或默认值 | `conn` 构造函数和 `validate_command` |
| `release()` 语义 | `process.cpp` 中 CONNECT/UDP/MUX 三条路径 |
| 缓冲区大小（320 字节） | 域名地址最大长度边界 |
| 超时时间（30 秒） | 所有握手场景的等待上限 |
