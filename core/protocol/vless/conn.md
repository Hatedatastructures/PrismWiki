---
layer: core
source:
  - "I:/code/Prism/include/prism/protocol/vless/conn.hpp"
  - "I:/code/Prism/src/prism/protocol/vless/conn.cpp"
  - "I:/code/Prism/src/prism/protocol/vless/framing.cpp"
title: VLESS 连接中继器
tags: [protocol, vless, conn, handshake, uuid, mux, udp, decorator]
---

# VLESS 连接中继器

> 源码位置: `I:/code/Prism/include/prism/protocol/vless/conn.hpp`
> 实现位置: `I:/code/Prism/src/prism/protocol/vless/conn.cpp`

## 模块概述

VLESS 连接中继器（`conn` 类）是 VLESS 协议的核心传输装饰器，负责协议握手、UUID 认证和命令分发。它继承 `transport::transmission` 接口，在底层传输层之上透明地添加 VLESS 协议语义，采用装饰器模式实现。

VLESS 协议运行在 TLS 内层，本身不提供加密。核心职责是通过 UUID 进行用户认证，并根据请求命令（TCP / UDP / MUX）提供不同的处理路径。

认证逻辑不直接耦合 `account::directory`，而是通过 `verifier` 回调委托给上层，由 `process` 入口函数绑定具体的账户查询逻辑。

## 类定义

```cpp
class conn : public psm::transport::transmission,
             public std::enable_shared_from_this<conn>
```

**继承关系**:
- `transport::transmission` — 统一传输层接口，所有 I/O 操作签名
- `std::enable_shared_from_this<conn>` — 协程中安全获取 `shared_ptr`

## 核心函数及职责

### 构造函数

```cpp
explicit conn(shared_transmission next_layer, const config &cfg = {},
              std::function<bool(std::string_view)> verifier = nullptr);
```

- `next_layer`: 底层传输层（通常是 TLS 加密流或 preview 包装器）
- `cfg`: UDP 开关和超时参数
- `verifier`: UUID 验证回调，为 `nullptr` 时跳过认证（允许所有连接）

### handshake

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
```

VLESS 协议握手的完整入口，四步流水线：

1. **parse_header** — 读取固定头部（26 字节），校验版本号、提取 UUID、命令、端口、地址类型
2. **process_target** — 根据地址类型（IPv4 / IPv6 / Domain）读取变长目标地址
3. **send_response** — 调用 `verifier` 验证 UUID，通过后发送 2 字节响应
4. **构建 `request`** — 组装完整的请求结构返回给调用者

**超时保护**: 30 秒 `steady_timer`，超时后调用 `next_layer_->cancel()` 中断阻塞读取。

### parse_header（私有）

```cpp
auto parse_header(std::array<std::uint8_t, 320> &buffer,
                  std::span<std::byte> byte_span,
                  net::steady_timer &deadline)
    -> net::awaitable<std::tuple<fault::code,
          std::array<std::uint8_t, 16>, command,
          std::uint16_t, address_type, std::size_t>>;
```

从传输层读取并解析 VLESS 请求的固定前缀。使用 320 字节栈缓冲区，足够容纳最大请求（domain=255 时约 278 字节）。

校验规则：
- 版本号必须为 `0x00`
- 附加信息长度（AddnlInfoLen）必须为 `0`（不支持 XTLS/Vision）
- 命令必须为 `tcp` / `udp` / `mux` 之一

### process_target（私有）

```cpp
auto process_target(target_opts opts)
    -> net::awaitable<std::tuple<fault::code, address, std::size_t>>;
```

解析变长目标地址。参数通过 `target_opts` 结构体收敛（5 个参数收敛为 1 个，遵循 Rule 1）。

地址读取策略：
- **IPv4**: 从 offset 22 直接读取 4 字节
- **IPv6**: 从 offset 22 直接读取 16 字节
- **Domain**: 先读取 1 字节长度前缀，再读取对应长度的域名内容

所有路径都可能需要追加读取（`read_remaining`），因为首次 `read_min` 仅保证 26 字节。

### send_response（私有）

```cpp
auto send_response(const std::array<std::uint8_t, 16> &uuid,
                   net::steady_timer &deadline)
    -> net::awaitable<fault::code>;
```

两步操作：
1. 调用 `verifier_(uuid_str)` 进行 UUID 验证，将 16 字节二进制 UUID 转换为 `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` 格式字符串
2. 写入 2 字节响应 `[0x00, 0x00]`（通过 `format::make_response()` 生成）

### async_associate

```cpp
auto async_associate(route_callback route_cb) const
    -> net::awaitable<fault::code>;
```

进入 UDP over TLS 模式。前置检查 `config_.enable_udp`，禁用时立即返回 `fault::code::not_supported`。

实际帧循环委托给 `protocol::common::frame_loop`，传入 VLESS 特定的解析/构建函数（`format::parse_udp_pkt` / `format::build_udp_pkt`）。

流量统计通过 `traffic_context` 结构体传递给循环，循环结束时通过回调触发 `flush_traffic`。

### 透传方法

`async_read_some`、`async_write_some`、`close`、`cancel`、`executor` 全部透传到 `next_layer_`。`conn` 本身不缓冲数据，握手完成后即变为透明管道。

### release

```cpp
auto release() -> shared_transmission;
```

释放底层传输层所有权。握手成功后，`process` 入口调用此方法将传输层转移给后续的转发或多路复用组件。

## 设计决策

### WHY: 装饰器模式包装传输层

**Problem**: VLESS 协议需要在已有的 TLS 传输层上添加握手和认证逻辑，但不能破坏统一的传输层接口。

**Choice**: `conn` 继承 `transport::transmission`，内部持有 `shared_transmission next_layer_`，所有 I/O 方法透传到底层。握手完成后通过 `release()` 将底层传输层转移给隧道转发组件。

**Consequence**: 协议处理逻辑与传输层完全解耦。同一个传输层可以被 VLESS、Trojan、SOCKS5 等不同协议装饰器包装。`release()` 后 `conn` 对象本身不再持有传输层，生命周期管理清晰。

### WHY: verifier 回调委托认证

**Problem**: 协议层不应直接依赖账户管理模块，且不同部署场景的认证策略可能不同。

**Choice**: 通过 `std::function<bool(std::string_view)>` 回调注入认证逻辑。`conn` 只负责将二进制 UUID 转为字符串并调用回调，不关心查询细节。

**Consequence**: `conn` 的测试可以注入简单的验证函数而无需模拟整个 `account::directory`。实际使用时由 `process` 入口绑定 `account::try_acquire` 查询。`verifier` 为 `nullptr` 时跳过认证，支持无认证部署。

### WHY: 320 字节栈缓冲区

**Problem**: VLESS 请求头最大长度约 278 字节（domain=255），需要在协程栈上分配足够空间。

**Choice**: 使用 `std::array<std::uint8_t, 320>` 栈分配，覆盖最大请求且无堆分配开销。

**Consequence**: 无 PMR 分配器依赖，适合高频握手路径。但 320 字节几乎没有余量——如果未来协议扩展增加字段，需要重新评估缓冲区大小。

### WHY: 2 字节响应而非 1 字节

**Problem**: 客户端（mihomo / Xray / sing-box）期望读取 2 字节响应。

**Choice**: `make_response()` 返回 `[Version 0x00, Addons Length 0x00]` 共 2 字节。

**Consequence**: 仅发送 1 字节会导致客户端将后续 smux ACK 数据误读为 Addons Length，造成流偏移。这是 VLESS 协议的已知兼容性陷阱。

## 约束

- **不支持 XTLS / Vision flow**: `addnl_info_len != 0` 时握手直接返回 `fault::code::bad_message`
- **握手超时硬编码 30 秒**: 无法通过配置调整
- **UDP 默认禁用**: 需要在 `config` 中显式启用 `enable_udp`
- **缓冲区无余量**: 320 字节仅比最大请求（278 字节）多 42 字节

## 失败场景

| 场景 | 错误码 | 行为 |
|------|--------|------|
| 版本号非 0x00 | `bad_message` | 取消超时定时器，握手失败 |
| AddnlInfoLen 非 0 | `bad_message` | 同上，不支持扩展信息 |
| 未知命令字 | `unsupported_command` | 同上 |
| 未知地址类型 | `unsupported_address` | 同上 |
| UUID 验证失败 | `auth_failed` | 记录警告日志，不发送响应 |
| 握手超时（30s） | `timeout` | 取消底层传输操作 |
| 底层读取错误 | 原始错误码 | 透传 `fault::to_code(ec)` |
| UDP 未启用 | `not_supported` | 立即返回，不进入帧循环 |
| 响应写入失败 | 写入错误码 | 透传底层 I/O 错误 |

## 跨模块契约

### 上游依赖

| 模块 | 契约 |
|------|------|
| `transport::transmission` | 底层传输接口，`conn` 持有其 `shared_ptr` |
| `transport::preview` | 握手前由 `process` 包装的预读回放层，`conn` 透传读取 |
| `transport::async_write` | 自由函数，用于写入响应，保证完整写入 |
| `protocol::common::read_min` | 批量读取工具，循环 `async_read_some` 直到满足最小字节数 |
| `protocol::common::read_remaining` | 追加读取工具，读取剩余需要的字节 |
| `protocol::common::frame_loop` | UDP 帧循环基础设施，VLESS 传入自己的解析/构建函数 |
| `vless::format` | 协议格式编解码（响应生成、UDP 帧解析/构建） |
| `vless::config` | UDP 开关和超时参数 |
| `stats::traffic::traffic_state` | 流量统计，通过 `set_traffic` 注入 |

### 下游消费者

| 模块 | 契约 |
|------|------|
| `vless::handle` (process) | 调用 `make_conn` 创建实例，执行 `handshake()`，根据命令分发 |
| `connect::forward` | TCP 命令路径，接收 `release()` 后的传输层进行隧道转发 |
| `multiplex::bootstrap` | MUX 命令路径，接收 `release()` 后的传输层进行多路复用协商 |
| `outbound::proxy` | UDP 命令路径，提供 `route_callback` 用于 DNS 解析 |

## 变更敏感度

| 修改 | 影响范围 |
|------|----------|
| 修改握手流程（如增加扩展字段解析） | `parse_header`、`process_target`、缓冲区大小 |
| 新增命令字 | `parse_header` 的 switch、`process` 入口的 switch |
| 修改 UUID 格式 | `send_response` 中的 `uuid_to_string`、`verifier` 签名 |
| 修改响应格式 | `format::make_response()`、客户端兼容性 |
| 修改超时时间 | `handshake()` 中的 `steady_timer` 构造参数 |
| 修改缓冲区大小 | 所有 `read_min` / `read_remaining` 调用的 span 大小 |

## 调用链

- [[core/protocol/vless/format|Format]] — 响应生成和 UDP 帧编解码
- [[core/protocol/vless/constants|Constants]] — 版本号和命令枚举
- [[core/protocol/vless/config|Config]] — UDP 开关和超时参数
- [[core/protocol/vless/process|Process]] — 调用 conn 的业务入口
- [[core/protocol/common/read|Read Utils]] — 共享读取工具函数
- [[core/protocol/common/udprelay|UDP Relay]] — 共享 UDP 帧循环
- [[core/transport/transmission|Transmission]] — 底层传输层接口
- [[core/transport/preview|Preview]] — 预读数据回放包装器
- [[core/account/entry|Account Entry]] — 连接租约和流量统计
