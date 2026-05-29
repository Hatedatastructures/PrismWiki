---
title: "transport -- 传输层总览"
layer: core
source: "include/prism/transport/"
module: "transport"
type: overview
tags: [transport, transmission, reliable, encrypted, unreliable, snapshot, preview, connector]
created: 2026-05-23
updated: 2026-05-23
related:
  - core/connect/overview
  - core/instance/overview
---

# transport -- 传输层总览

> 源码目录: `include/prism/transport/`
> 模块: [[core/transport/overview|transport]]
> 定位: 请求处理流程的传输抽象层

## 模块职责

传输层（Transport Layer）提供统一的网络读写抽象，封装 TCP socket、TLS 流、UDP socket 等底层传输机制。所有协议层的读写操作都通过传输层接口完成，实现协议与底层传输的解耦。传输层是 Prism 协程架构的基础设施层，所有异步 I/O 操作最终都通过传输层完成。

核心职责：

| 职责 | 说明 | 关键组件 |
|------|------|----------|
| 传输层抽象 | 统一 TCP/UDP/TLS 的读写接口 | [[core/transport/transmission|transmission]] |
| TCP 可靠传输 | 封装 TCP socket，支持连接池复用 | [[core/transport/reliable|reliable]] |
| TLS 加密传输 | 封装 TLS 流，提供加密读写 | [[core/transport/encrypted|encrypted]] |
| UDP 数据报传输 | 封装 UDP socket，模拟连接式操作 | [[core/transport/unreliable|unreliable]] |
| 预读数据回放 | 将探测阶段的预读数据注入后续读取流 | [[core/transport/preview|preview]] |
| 可回滚读取 | 捕获读取数据支持 rewind 重试 | [[core/transport/snapshot|snapshot]] |
| Socket 适配器 | 适配 Boost.Asio AsyncStream 概念 | [[core/transport/adapter/connector|connector]] |

## 源码结构

```
transport/
├── transmission.hpp          # 抽象接口基类 + 组合操作自由函数
├── reliable.hpp              # TCP 可靠传输（叶子节点）
├── encrypted.hpp             # TLS 加密传输（叶子节点，持有 ssl::stream）
├── unreliable.hpp            # UDP 数据报传输（叶子节点）
├── snapshot.hpp              # 可回滚读取装饰器
├── preview.hpp               # 预读数据回放装饰器
└── adapter/
    └── connector.hpp         # Boost.Asio AsyncStream 适配器
```

## transmission 抽象接口

[[core/transport/transmission|transmission]] 是所有传输实现的抽象基类，采用纯协程设计。

### 核心虚函数

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `transport_type()` | `type` | 返回 TCP 或 UDP 类型标识 |
| `executor()` | `executor_type` | 返回关联的执行器 |
| `async_read_some(buffer, ec)` | `awaitable<size_t>` | 异步读取部分数据 |
| `async_write_some(buffer, ec)` | `awaitable<size_t>` | 异步写入部分数据 |
| `close()` | `void` | 关闭传输层 |
| `cancel()` | `void` | 取消所有未完成的异步操作 |
| `next_layer()` | `transmission*` | 装饰器链导航，叶子节点返回 nullptr |

### 传输类型标识

```cpp
enum class type
{
    tcp,  // 可靠流式传输（TCP）
    udp   // 不可靠数据报传输（UDP）
};
```

`transport_type()` 允许协议层在运行时查询底层传输的真实类型，装饰器沿 `next_layer()` 链委托到底层叶子节点。

### Completion-handler 双路径

transmission 提供两套异步读取/写入接口：

1. **协程路径** (`awaitable`): 返回 `net::awaitable<size_t>`，通过 `ec` 参数返回错误码
2. **回调路径** (`any_completion_handler`): 直接委托给底层 socket，消除中间协程帧开销

叶子节点（reliable）覆写回调路径直接委托给 TCP socket，实现零协程帧的性能优势。这对 BoringSSL `ssl::stream` 的内部读写至关重要。

### 组合操作自由函数

基类不提供组合操作，通过自由函数实现：

```cpp
// 完整写入（循环调用 async_write_some 直到所有数据发送）
auto async_write(transmission &t, span<const byte> buffer, error_code &ec)
    -> net::awaitable<size_t>;

// 完整读取（循环调用 async_read_some 直到缓冲区填满）
auto async_read(transmission &t, span<byte> buffer, error_code &ec)
    -> net::awaitable<size_t>;
```

### lowest_layer 链式导航

```cpp
template <typename T>
T *lowest_layer() noexcept;
```

沿 `next_layer()` 走到链底，然后 `dynamic_cast` 为目标类型。替代所有手动 `dynamic_cast<preview*>` / `dynamic_cast<snapshot*>` 的剥壳循环。

## 传输类型详解

### reliable -- TCP 可靠传输

[[core/transport/reliable|reliable]] 封装 `boost::asio::ip::tcp::socket`，是传输层最核心的叶子节点实现。

**三种构造方式**:
1. `reliable(executor)` -- 从执行器创建空 socket
2. `reliable(socket)` -- 从已连接 socket 创建
3. `reliable(pooled_connection)` -- 从 [[core/connect/pool/pool|连接池]]获取的连接创建

**连接池复用**: 当通过 `pooled_connection` 构造时，`close()` 不直接关闭 socket，而是将其归还到连接池。池连接的健康状态由 `healthy_fast()` 检测。

**独有方法**:
- `shutdown_write()` -- TCP 半关闭（写方向）
- `native_socket()` -- 直接访问底层 TCP socket
- `release_socket()` -- 释放 socket 所有权（用于 ShadowTLS/Restls 接管）

### encrypted -- TLS 加密传输

[[core/transport/encrypted|encrypted]] 将 `ssl::stream<connector>` 适配为 `transmission` 接口，提供 TLS 加密读写。

**核心职责**: 将 BoringSSL 的 SSL 流适配为统一的传输层接口，使协议装饰器（如 [[core/protocol/trojan/format|trojan::conn]]）能够透明地装饰 TLS 加密流。

**静态工厂方法**:
```cpp
static auto ssl_handshake(shared_transmission inbound, ssl::context &ssl_ctx)
    -> net::awaitable<tuple<fault::code, shared_stream, shared_transmission>>;
```
将入站传输层包装为 connector，执行 TLS 服务端握手。握手失败时从 connector 释放传输层所有权，避免传输层丢失。

**关闭行为**: 先发送 TLS `close_notify` 通知对端，然后关闭底层传输。

### unreliable -- UDP 数据报传输

[[core/transport/unreliable|unreliable]] 封装 `boost::asio::ip::udp::socket`，在无连接的 UDP 上模拟连接式操作。

**连接模拟**: 内部维护一个 `remote_endpoint_`，所有发送操作指向该端点，接收操作自动过滤非远程端点的数据报（不匹配则丢弃并继续等待）。

**自动端点学习**: 如果未设置远程端点，首次接收的数据报来源将自动设为远程端点。

### preview -- 预读数据回放

[[core/transport/preview|preview]] 是装饰器，将协议探测阶段预读的数据注入后续读取流。

**使用场景**: [[core/recognition/probe/probe|probe]] 阶段预读 24 字节用于协议检测，但这些数据需要回放给后续的协议处理器。`preview` 将预读数据保存在内部缓冲区，后续读取时优先返回预读数据，耗尽后委托给内层传输。

```cpp
auto inbound = transport::wrap_with_preview(std::move(ctx.inbound), data, arena);
```

### snapshot -- 可回滚读取

[[core/transport/snapshot|snapshot]] 是装饰器，自动捕获所有从内层读取的字节，支持 `rewind()` 回到起点重新读取。

**使用场景**: [[core/stealth/overview|TLS 伪装方案]]的依次尝试。每个 scheme 读取的数据被 snapshot 捕获，失败时 `rewind()`，下一个 scheme 从同一起点重试。

**设计约束**:
- `rewind()` 仅在未发生写入时有效（`wrote_ == false`）
- 认证阶段是纯读取，安全 rewind
- 一旦开始写入（如 TLS 握手），传输层状态不可恢复

### connector -- Asio 适配器

[[core/transport/adapter/connector|connector]] 将 `transmission` 接口适配为 Boost.Asio 的 `AsyncReadStream`/`AsyncWriteStream` 概念，使得 `ssl::stream<connector>` 可以包装任意 transmission。

**预读数据注入**: connector 支持在构造时注入预读数据，首次 `async_read_some` 优先从预读缓冲区返回。

## 传输层栈式包装

传输层支持多层装饰器包装，形成栈式结构：

### 典型栈组合

**TLS 代理（如 Trojan over TLS）**:
```
trojan::conn (协议装饰器)
  └─ encrypted (TLS 加密)
       └─ connector (Asio 适配)
            └─ preview (预读回放)
                 └─ reliable (TCP socket)
```

**Reality 伪装方案**:
```
reality::seal (TLS 1.3 加密传输层)
  └─ snapshot (可回滚读取)
       └─ reliable (TCP socket)
```

**UDP over TLS（Trojan UDP_ASSOCIATE）**:
```
trojan::conn (协议装饰器)
  └─ encrypted (TLS 加密)
       └─ connector (Asio 适配)
            └─ reliable (TCP socket)
```

**普通 HTTP 代理**:
```
http::conn (协议处理器，非传输装饰器)
  └─ preview (预读回放)
       └─ reliable (TCP socket)
```

### 链式导航

通过 `lowest_layer<T>()` 穿透所有包装层获取指定类型的底层传输：

```cpp
auto *tcp = transport->lowest_layer<transport::reliable>();
// 从 trojan::conn → encrypted → connector → preview → reliable
```

## 模块依赖关系

```
transport/
├── transmission     ← 抽象基类，所有传输实现的父类
├── reliable         → 依赖 pool (pooled_connection)、fault (错误码映射)
├── encrypted        → 依赖 connector、fault、trace、BoringSSL
├── unreliable       → 依赖 fault (错误码映射)，无其他内部依赖
├── preview          → 依赖 transmission、memory (PMR 容器)
├── snapshot         → 依赖 transmission、memory (PMR 容器)
└── adapter/connector → 依赖 transmission、memory (PMR 容器)
```

## 调用链

### 上游调用（入）

| 上游模块 | 调用入口 | 说明 |
|----------|----------|------|
| [[core/protocol/http/relay|Protocol HTTP]] | `transport::wrap_with_preview()` | HTTP 处理前包装预读数据 |
| [[core/protocol/trojan/format|Protocol Trojan]] | `transmission::async_read_some()` | Trojan 协议读写使用传输层接口 |
| [[core/protocol/socks5/stream|Protocol SOCKS5]] | `transmission::async_write_some()` | SOCKS5 协议读写使用传输层接口 |
| [[core/stealth/reality/seal|Reality Seal]] | `snapshot::rewind()` | Reality 加密传输使用 snapshot |
| [[core/multiplex/duct|Multiplex Duct]] | `transport::async_write()` | 多路复用帧写入 |
| [[core/transport/overview|Transport]] | `transport::async_read/write()` | 双向隧道数据转发 |

### 下游调用（出）

| 下游模块 | 调用出口 | 说明 |
|----------|----------|------|
| Boost.Asio | `tcp::socket::async_read_some()` | TCP 数据收发 |
| BoringSSL | `ssl::stream::async_read_some()` | TLS 加密数据收发 |
| Boost.Asio | `udp::socket::async_receive_from()` | UDP 数据报收发 |
| [[core/connect/pool/pool|Connection Pool]] | `pooled_connection` | 连接池复用管理 |

## 设计决策

### 为什么使用 transmission 基类 + 装饰器链而非继承体系？

**问题**: 传输层有多种变体（TCP、TLS、UDP）和多种功能横切（预读回放、可回滚、协议装饰），如果用继承体系会组合爆炸。

**选择**: transmission 基类定义最小流接口（async_read_some、async_write_some、executor、close），装饰器（preview、snapshot、protocol conn）通过组合包装任意 transmission。叶子节点（reliable、encrypted、unreliable）实现具体传输。

**后果**: 任意传输都可以被任意装饰器包装，如 `preview(snapshot(reliable))` 或 `encrypted(connector(preview(reliable)))`。代价是 `next_layer()` 链式导航需要 O(N) 遍历，但装饰层通常不超过 4 层，开销可忽略。

**替代方案**: 为每种组合定义类型（如 `TLSOverTCPWithPreview`）会导致笛卡尔积爆炸。

**源码依据**: `transport/transmission.hpp:57-237`

### 为什么组合操作是自由函数而非虚函数？

**问题**: async_write（完整写入）和 async_read（完整读取）是循环调用 async_write_some/read_some 的组合操作，如果作为虚函数则每个子类都要覆写。

**选择**: 作为自由函数 `transport::async_write()` / `transport::async_read()` 定义在 transmission.hpp 中，所有传输类型共享同一实现。

**后果**: 减少虚函数数量，子类只需实现 `async_read_some` 和 `async_write_some`。UDP 的 `async_write` 行为与 TCP 不同时，由 unreliable 自身在虚函数覆写中处理（UDP 数据报一次发送完成无需循环）。

**源码依据**: `transport/transmission.hpp:255-296`

### 为什么 reliable 和 unreliable 都继承 enable_shared_from_this？

**问题**: 连接池归还和异步操作生命周期需要 shared_ptr 管理，但构造函数中无法获取自身的 shared_ptr。

**选择**: reliable 和 unreliable 都继承 `std::enable_shared_from_this`，允许在成员函数中安全获取 `shared_from_this()`。

**后果**: 对象必须通过 `std::make_shared` 创建（工厂函数保证）。栈上创建或 `std::unique_ptr` 管理的对象调用 `shared_from_this()` 是未定义行为。

**源码依据**: `transport/reliable.hpp:52`, `transport/unreliable.hpp:41`

## 约束

### 装饰器链顺序固定

**类型**: 架构约束
**规则**: 装饰器链顺序由外到内为: protocol conn -> encrypted -> connector -> preview -> snapshot -> reliable。snapshot 必须在 preview 之下（更接近 reliable），否则 rewind 无法回放 preview 已消费的预读数据。
**违反后果**: rewind 后丢失预读数据，协议处理读到残缺数据流

### 所有异步操作必须返回 net::awaitable

**类型**: 协程纯度
**规则**: transmission 的虚函数 async_read_some/async_write_some 返回 `net::awaitable<size_t>`，错误通过 `std::error_code&` 参数返回，不抛异常。
**违反后果**: 在协程中抛异常会违反热路径无异常保证

### close() 后禁止继续使用

**类型**: 生命周期
**规则**: close() 关闭底层资源后，传输层对象不再可用。调用方不应在 close() 后调用任何方法。

## 失败场景

| 场景 | 处理 |
|------|------|
| TCP 连接断开 | async_read_some 返回 ec=connection_reset 或 n=0 |
| TLS 握手失败 | encrypted::ssl_handshake 返回 fault::code，从 connector 恢复 transport |
| UDP 数据报来源不匹配 | unreliable 丢弃数据报，循环等待匹配来源 |
| snapshot rewind 后写入 | can_rewind() 返回 false，调用方应检查 |
| preview 预读数据耗尽 | 透明委托给 inner_->async_read_some，无额外开销 |
| 连接池连接不健康 | healthy_fast() 检测后自动销毁而非复用 |

## 跨模块契约

| 上游模块 | 对 transport 的调用 | 契约 |
|----------|---------------------|------|
| recognition | wrap_with_preview(inbound, probe_data) | probe 阶段预读数据必须在协议处理前包装回传输 |
| stealth | make_snapshot(inbound) + rewind() | rewind 仅在纯读取阶段有效 |
| connect/tunnel | async_write(t, buffer, ec) | 完整写入保证所有数据发送或返回错误 |
| multiplex | transport->async_read_some() | 传输层作为 mux frame_loop 的数据源 |
| connect/pool | make_reliable(pooled_connection) | 池连接的 close() 归还而非关闭 |

## 参见

- [[core/transport/transmission|transmission]] -- 传输层抽象接口
- [[core/transport/reliable|reliable]] -- TCP 可靠传输
- [[core/transport/encrypted|encrypted]] -- TLS 加密传输
- [[core/transport/unreliable|unreliable]] -- UDP 数据报传输
- [[core/transport/snapshot|snapshot]] -- 可回滚读取装饰器
- [[core/transport/preview|preview]] -- 预读数据回放装饰器
- [[core/transport/adapter/connector|connector]] -- Asio 适配器
- [[core/connect/overview|connect]] -- 连接层总览
- [[core/connect/pool/pool|连接池]] -- 连接复用管理
- [[core/instance/overview|instance]] -- 实例层总览

## 故障场景

### 1. TLS 握手失败

**触发条件**: 客户端证书不匹配、SNI 无后端、BoringSSL 内部错误
**传播路径**: `encrypted::ssl_handshake()` → 返回 `fault::code` → 从 `connector` 释放传输层所有权 → 调用方决定终止或走 native 兜底
**外部表现**: 客户端收到 TLS alert，连接中断
**恢复机制**: 客户端需重新发起连接，无法在当前连接上恢复

### 2. TCP 连接中断

**触发条件**: 网络超时、对端 RST、中间设备丢包
**传播路径**: `reliable::async_read_some()` 返回 `ec=connection_reset` 或 `n=0` → 上层协议处理器退出读写循环 → session 关闭
**外部表现**: 数据传输停止，双向转发终止
**恢复机制**: 无，连接不可恢复；连接池中该 socket 被 `healthy_fast()` 检测后销毁

### 3. 装饰器链 rewind 失败

**触发条件**: `snapshot` 包装的传输层在读取阶段之后发生写入（`wrote_==true`），调用 `rewind()`
**传播路径**: `can_rewind()` 返回 false → stealth 执行器跳过 rewind → 管道终止或走 native 兜底
**外部表现**: 当前伪装方案失败后无法尝试下一个方案
**恢复机制**: native 兜底方案作为最终回退

### 4. 连接池连接不健康

**触发条件**: 池中复用的 TCP 连接已被对端关闭或处于半开状态
**传播路径**: `healthy_fast()` 探测失败 → 连接从池中移除并销毁 → 调用方重新拨号
**外部表现**: 首次使用池连接时短暂延迟（探测 + 重连），不影响正确性
**恢复机制**: 自动重连，调用方透明

### 5. UDP 数据报端点不匹配

**触发条件**: `unreliable` 收到来自非预期远程端点的数据报
**传播路径**: 数据报被静默丢弃 → 循环等待匹配来源 → 可能无限等待
**外部表现**: 如果仅有不匹配来源的数据报，数据流挂起
**恢复机制**: 依赖对端重传或超时机制

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| `transmission` 虚函数签名变更 | 所有叶子节点（reliable/encrypted/unreliable）和装饰器（preview/snapshot） | 必须同步更新所有实现，否则编译失败或运行时 UB |
| `shared_transmission` 类型定义变更 | 整个协议栈（protocol/stealth/connect/multiplex） | 所有持有传输层指针的模块需修改 |
| 装饰器链构造顺序变更 | recognition、stealth、protocol 层 | rewind/preview 语义变化，可能导致预读数据丢失 |
| `async_write`/`async_read` 自由函数行为变更 | 所有使用组合写入/读取的模块 | 帧完整性或数据丢失 |
| `close()` 语义变更（如连接池归还逻辑） | connect/pool、session 生命周期 | 池连接泄漏或过早释放 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| Boost.Asio `AsyncStream` 概念变更 | `connector` 适配器实现 | `async_read_some`/`async_write_some` 签名兼容性 |
| BoringSSL `ssl::stream` API 变更 | `encrypted` TLS 握手和关闭流程 | `ssl_handshake` 工厂方法、`close_notify` 发送逻辑 |
| `memory` 模块 PMR 容器类型变更 | `preview`、`snapshot`、`connector` 内部缓冲区 | 分配器类型、构造函数参数 |
| `fault::code` 枚举新增或重命名 | 所有传输类型的错误码映射 | `to_ec` 转换函数、错误路径分支 |
