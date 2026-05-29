---
title: SOCKS5 协议中继器
layer: core
module: protocol/socks5
source:
  - I:/code/Prism/include/prism/protocol/socks5/conn.hpp
  - I:/code/Prism/src/prism/protocol/socks5/conn.cpp
tags: [protocol, socks5, conn]
---

# SOCKS5 协议中继器

`socks5::conn` 是 SOCKS5 协议的服务端中继器，实现 RFC 1928 全部服务端逻辑：方法协商、认证、请求解析、命令检查、响应生成和 UDP 关联。它继承 `transport::transmission` 接口，作为装饰器包装底层传输层，对外提供协程友好的高层 API。

## 模块组成

| 组件 | 功能 | 行号范围 |
|------|------|----------|
| `conn` 类 | 核心中继器，持有传输层所有权 | conn.hpp:82-519 |
| `relay_context` | UDP 中继上下文，收敛多引用参数 | conn.hpp:57-63 |
| `make_conn()` | 工厂函数，封装 `make_shared` | conn.hpp:536-540 |
| `route_callback` | UDP 路由回调类型定义 | conn.hpp:48 |

## 核心函数

### handshake

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
```

执行完整 SOCKS5 握手流程，包含五个阶段：

1. **超时保护**：创建 30 秒 `steady_timer`，超时后 `cancel()` 底层传输
2. **方法协商**：`negotiated_authentication()` 读取客户端方法列表，选择认证方式
3. **请求头部**：`read_req_hdr()` 读取 4 字节 VER+CMD+RSV+ATYP
4. **命令检查**：`resolve_command()` 根据 `config` 验证命令是否允许
5. **地址解析**：`resolve_address()` 根据 ATYP 读取 IPv4/IPv6/域名 + 端口

成功时返回 `request` 结构体（含 `cmd`、`destination_address`、`destination_port`、`transport` form），失败时返回对应 `fault::code`。

### negotiated_authentication

```cpp
auto negotiated_authentication() -> net::awaitable<std::pair<fault::code, auth_method>>;
```

方法协商策略：

| 配置 | 客户端支持密码认证 | 选择 |
|------|-------------------|------|
| `enable_auth=true` + 有账户目录 | 是 | `password` (0x02)，触发 `password_auth()` |
| `enable_auth=true` + 有账户目录 | 否 | `no_acceptable` (0xFF)，拒绝连接 |
| `enable_auth=false` | 支持 `no_auth` | `no_auth` (0x00) |
| `enable_auth=false` | 不支持 `no_auth` | `no_acceptable` (0xFF) |

### password_auth

```cpp
auto password_auth() -> net::awaitable<std::pair<fault::code, bool>>;
```

RFC 1929 用户名/密码认证子协商：

1. 读取 VER(1) + ULEN(1) + UNAME(n) + PLEN(1) + PASSWD(n)，最大 513 字节
2. 解析为 `wire::pw_auth_request`（验证子协商版本 0x01、长度合法）
3. 对密码调用 `crypto::sha224()` 哈希
4. 通过 `account::try_acquire()` 查找凭证并获取连接租约
5. 认证失败时发送 `auth_result::failed` 响应，不暴露具体原因

### async_associate

```cpp
auto async_associate(const request &request_info, route_callback route_callback) const
    -> net::awaitable<fault::code>;
```

UDP_ASSOCIATE 完整处理流程：

```
bind_datagram_port()          // 绑定本地 UDP 端口
    ↓
send_assoc_ok()               // 发送 SOCKS5 成功响应（含绑定地址/端口）
    ↓
┌──────────────────────────────────────────────┐
│ associate_loop() || wait_ctrl_close()        │
│  ┌─ 空闲超时退出                              │
│  ├─ async_receive_from → relay_datagram()    │
│  └─ 控制面读 EOF → cancel ingress → 退出     │
└──────────────────────────────────────────────┘
    ↓
flush_traffic()               // 刷写 UDP 流量统计
```

关键特性：
- **空闲超时**：每次 `async_receive_from` 前重置定时器，超时退出循环
- **控制面关闭**：TCP 控制连接任意读结束即关闭 UDP 关联（RFC 1928 标准终止机制）
- **惰性出站 socket**：`egress_socket` 在循环外创建，首次使用时 `open()`，跨数据报复用
- **原子流量统计**：`udp_uplink_` / `udp_downlink_` 使用 `std::atomic`，在关联结束时刷写

### relay_datagram

```cpp
auto relay_datagram(relay_context ctx, std::span<const std::byte> ingress_packet,
                    const net::ip::udp::endpoint &client_endpoint) const
    -> net::awaitable<void>;
```

单个 UDP 数据报的完整转发流程：

1. `wire::decode_hdr()` 解码 SOCKS5 UDP 报头，获取目标地址/端口
2. `to_string()` 将地址转为字符串，`route_cb()` 查询目标端点
3. 通过 `egress_socket.async_send_to()` 发送 payload 到目标
4. `egress_socket.async_receive_from()` 等待回包
5. `wire::encode_dgram()` 将回包重新封装为 SOCKS5 UDP 数据报
6. `ingress_socket.async_send_to()` 回写客户端

`relay_context` 将 ingress/egress socket、路由回调和目标缓冲区聚合为单一结构，避免 `relay_datagram` 参数超过 3 个。

## 传输层装饰器

`conn` 继承 `transport::transmission`，实现装饰器模式：

```
transport::transmission (接口)
        ↑
    socks5::conn (装饰器)
        ↓ next_layer_
transport::transmission (底层 TLS/TCP)
```

- `async_read_some()` / `async_write_some()`：直接透传到底层
- `close()` / `cancel()`：委托给底层
- `next_layer()`：返回底层传输指针（装饰器链导航）
- `release()`：转移底层所有权，转移后 `is_valid()` 返回 false
- `underlying()`：获取底层引用（跳过装饰器链）

## 设计决策

### 为什么 conn 持有传输层独占所有权？

**问题**: SOCKS5 握手结束后，底层传输层需要同时支持协议读取和后续隧道转发，所有权不清晰会导致悬挂指针。

**选择**: `conn` 通过 `shared_transmission next_layer_` 持有独占所有权，握手机阶段通过 `async_read_some`/`send_impl` 读写，CONNECT 成功后通过 `release()` 转移给隧道模块。

**后果**: `conn` 的生命周期与底层连接绑定，`release()` 后对象进入无效状态但不会崩溃。调用方需在 `release()` 后不再调用读写方法。

### 为什么 UDP_ASSOCIATE 使用 `operator||` 并发组合？

**问题**: UDP_ASSOCIATE 需要同时处理数据面（UDP 数据报转发）和控制面（TCP 连接断开检测），任一完成即应终止整个关联。

**选择**: 使用 `boost::asio::experimental::awaitable_operators::operator||` 将 `associate_loop` 和 `wait_ctrl_close` 并发运行。

**后果**: 当 TCP 控制连接断开时，`wait_ctrl_close` 取消并关闭 ingress socket，驱动 `associate_loop` 快速退出。无需额外同步原语。两个协程在同一个 `io_context` 上运行，无需锁。

### 为什么使用栈分配缓冲区而非动态分配？

**问题**: SOCKS5 握手涉及多种长度的报文（方法协商最大 258 字节、认证最大 513 字节、请求头部固定 4 字节、响应最大 262 字节），动态分配会增加延迟。

**选择**: 所有内部缓冲区使用 `std::array<uint8_t, N>` 栈分配：`methods_buffer[258]`、`auth_buffer[513]`、`request_header[4]`、`response[262]`。

**后果**: 热路径零堆分配，但 `conn` 对象在栈上的占用较大（约 800+ 字节）。通过 `make_shared` 分配在堆上，避免栈溢出风险。

### 为什么认证使用 SHA224 而非明文比较？

**问题**: RFC 1929 认证以明文传输用户名和密码，服务端需要安全地存储和验证凭证。

**选择**: 使用 `crypto::sha224()` 对密码进行哈希后，通过 `account::try_acquire()` 在账户目录中查找匹配的哈希值。账户目录中预存的是 SHA224 哈希值。

**后果**: 密码不以明文存储在配置中，但网络传输仍为明文（需依赖 TLS 传输层保护）。认证失败不暴露具体原因（用户不存在 vs 密码错误），防止用户枚举攻击。

## 约束

### 传输层有效性

**类型**: 前置条件

**规则**: 调用 `async_read_some`、`async_write_some`、`handshake`、`send_success`、`send_error` 前必须确保 `is_valid()` 返回 true

**违反后果**: 解引用空指针，未定义行为

**源码依据**: conn.hpp:262-265

### 单线程使用

**类型**: 并发约束

**规则**: `conn` 实例非线程安全，必须在同一协程或 strand 内使用

**违反后果**: 数据竞争，`next_layer_` 并发读写导致协议状态错乱

**源码依据**: conn.hpp:76

### 配置不可变

**类型**: 生命周期约束

**规则**: `config_` 在构造时传入，运行时只读

**违反后果**: 如果外部修改 config 引用的对象，握手过程中命令检查结果可能不一致

**源码依据**: conn.hpp:512

## 故障场景

### 握手超时

**触发条件**: 客户端在 30 秒内未完成握手（网络延迟、恶意连接）

**传播路径**: `steady_timer` 触发 → `next_layer_->cancel()` → 正在进行的 `recv_impl` 返回 `operation_aborted` → 转换为 `fault::code::timeout`

**外部表现**: 客户端连接建立后 30 秒内无响应则断开

**恢复机制**: 自动回收，等待下一个连接

**日志关键字**: 无（process.cpp 仅记录 `handshake failed`，不区分超时）

### 认证失败

**触发条件**: 启用认证但客户端凭证无效（用户名不存在、密码错误、并发连接数超限）

**传播路径**: `password_auth()` → `account::try_acquire()` 返回空 lease → 发送 `auth_result::failed` → `negotiated_authentication()` 返回 `fault::code::success` + `auth_method::no_acceptable`

**外部表现**: 客户端收到认证失败响应，连接关闭

**恢复机制**: 客户端可重新尝试

**日志关键字**: 无（认证失败不记录日志，防止暴力破解信息泄露）

### UDP 端口绑定失败

**触发条件**: 系统端口耗尽或权限不足

**传播路径**: `bind_datagram_port()` → `socket.open()` 或 `socket.bind()` 返回错误 → `send_error(reply_code::server_failure)` → 返回 `fault::to_code(ec)`

**外部表现**: 客户端收到 server_failure 响应

**恢复机制**: 系统管理员检查端口占用和防火墙规则

### UDP 关联空闲超时

**触发条件**: 客户端在 `config_.idle_timeout` 秒内未发送任何 UDP 数据报

**传播路径**: `associate_loop` 中 `idle_timer.async_wait` 先于 `async_receive_from` 完成 → 协程退出 → 刷写流量统计 → 关联结束

**外部表现**: UDP 关联自动关闭

**恢复机制**: 客户端需重新建立关联

### 控制面异常断开

**触发条件**: TCP 控制连接断开（客户端崩溃、网络中断）

**传播路径**: `wait_ctrl_close()` 中 `async_read_some` 返回 EOF 或错误 → `ingress_socket.cancel()` + `close()` → `associate_loop` 中 `async_receive_from` 返回 `operation_aborted`

**外部表现**: UDP 关联立即终止

## 跨模块契约

| 模块 A | 模块 B | 契约内容 |
|--------|--------|---------|
| `conn` | [[core/protocol/socks5/wire\|wire]] | `conn` 调用 `wire::decode_hdr()` 和 `wire::encode_dgram()` 处理 UDP 数据报，依赖 `hdr_parse.header_size` 正确指向 DATA 起始偏移 |
| `conn` | [[core/protocol/socks5/constants\|constants]] | `conn` 使用 `reply_code` 枚举构建错误响应，值必须与 RFC 1928 REP 字段一一对应 |
| `conn` | [[core/protocol/socks5/config\|config]] | `conn` 在 `resolve_command()` 中检查 `enable_tcp`/`enable_udp`/`enable_bind`，配置变更需重启生效 |
| `conn` | [[core/crypto/sha224\|sha224]] | `password_auth()` 使用 `crypto::sha224()` 哈希密码，必须与 `account::directory` 中存储的哈希算法一致 |
| `conn` | [[core/account/directory\|account]] | `password_auth()` 通过 `account::try_acquire()` 获取连接租约，认证成功后持有 lease 直到会话结束 |
| `conn` | [[core/transport/transmission\|transmission]] | `conn` 继承 `transmission` 接口，`async_read_some`/`async_write_some` 的签名必须与基类一致 |
| `conn` | [[core/connect/tunnel\|tunnel]] | CONNECT 成功后 `conn` 通过 `release()` 转移传输层给 `connect::tunnel()` 进行双向转发 |
| `process` | [[core/transport/preview\|preview]] | `process` 通过 `wrap_with_preview()` 将预读数据回放到传输层，确保 SOCKS5 握手看到完整数据流 |
| `process` | [[core/connect/dial\|dial]] | `process` 调用 `connect::dial()` 建立上游连接，拨号失败映射为 `reply_code::host_unreachable` |
| `process` | [[core/outbound/proxy\|proxy]] | `process` 优先使用 `outbound_proxy` 拨号，其次使用 `router` 路由 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| 修改 `handshake()` 返回类型 | `process.cpp` 及所有调用方 | 编译失败 |
| 修改 `request` 结构体字段 | `process.cpp`、所有命令分发逻辑 | 编译失败 |
| 修改 `route_callback` 签名 | `outbound::router_fn`、`async_associate` | 编译失败 |
| 修改 `send_success()` 语义 | 连接隧道建立时序 | 上游连接建立后才能发送成功响应 |
| 修改 `release()` 语义 | `connect::tunnel()` 入参 | 隧道获取不到有效传输层 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| `wire::decode_hdr()` 返回结构变化 | `relay_datagram` | `hdr_parse` 字段访问 |
| `account::try_acquire()` 接口变化 | `password_auth` | lease 获取和持有逻辑 |
| `transport::transmission` 虚函数签名变化 | `conn` 所有 override | 编译错误 |
| `config` 结构字段增减 | `resolve_command`、`bind_datagram_port`、`associate_loop` | 命令检查和 UDP 参数 |
| `relay_context` 字段变化 | `relay_datagram`、`associate_loop` | 参数传递和 socket 复用 |

## 实现边界

- **UDP ASSOCIATE 仅绑定 IPv4**：`bind_datagram_port()` 中 `udp::v4()` 硬编码，IPv6 客户端的 UDP ASSOCIATE 将静默失败
- **BIND 命令有条件支持**：代码路径存在但默认禁用（`enable_bind=false`），未经过完整测试
- **握手超时硬编码**：30 秒超时值在 `handshake()` 内部硬编码，不通过 `config` 配置
- **UDP 出站 socket 不绑定**：`egress_socket` 仅 `open()` 不 `bind()`，由系统自动分配源端口

## 相关文档

- [[core/protocol/socks5/process|process]] - SOCKS5 协议入口函数
- [[core/protocol/socks5/constants|constants]] - 协议常量定义
- [[core/protocol/socks5/config|config]] - 协议配置结构
- [[core/protocol/socks5/wire|wire]] - 线级编解码
- [[core/transport/transmission|transmission]] - 传输层抽象接口
- [[core/connect/tunnel|tunnel]] - 双向隧道转发
- [[core/crypto/sha224|sha224]] - 密码哈希
