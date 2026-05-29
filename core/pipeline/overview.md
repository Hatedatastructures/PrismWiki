---
title: "Pipeline 层 -- 协议管道处理"
layer: core
module: pipeline
source: include/prism/pipeline/
created: 2026-05-17
updated: 2026-05-27
tags: [pipeline, overview, protocol-handler, dispatch, tunnel]
---

# Pipeline 层 — 协议管道处理

Pipeline 层是 Prism 六阶段架构的第四阶段，位于 Dispatch 层和 Channel 层之间。核心职责：预读重放、协议握手、目标解析、上游拨号、双向转发。

## 设计决策

### 为什么每个协议是独立函数而非继承基类？

协议处理器的逻辑差异大（HTTP 解析请求行 vs SOCKS5 解析握手帧 vs Trojan 读 hex 密码），无法抽象为统一接口。独立函数 + `switch` 分发更直接，编译器可内联优化。Dispatch 层通过 `protocol_type` 枚举选择对应函数。

**后果**: 新增协议只需添加新函数和 switch case，不涉及继承层次。

## 模块组成

| 组件 | 文件 | 说明 |
|------|------|------|
| [[core/pipeline/primitives\|primitives]] | `primitives.hpp` | 管道原语（shut_close/dial/tunnel/forward） |
| [[core/pipeline/protocols/http\|http]] | `protocols/http.hpp` | HTTP 代理处理 |
| [[core/pipeline/protocols/socks5\|socks5]] | `protocols/socks5.hpp` | SOCKS5 代理处理 |
| [[core/pipeline/protocols/trojan\|trojan]] | `protocols/trojan.hpp` | Trojan 协议处理 |
| [[core/pipeline/protocols/vless\|vless]] | `protocols/vless.hpp` | VLESS 协议处理 |
| [[core/pipeline/protocols/shadowsocks\|shadowsocks]] | `protocols/shadowsocks.hpp` | SS2022 协议处理 |
| [[core/pipeline/protocols/http|HTTP]] | `http()` | 外层可选 | Basic Auth 或无 |
| [[core/pipeline/protocols/socks5|SOCKS5]] | `socks5()` | 无 | 方法协商 |
| [[core/pipeline/protocols/trojan|Trojan]] | `trojan()` | Session 层 | SHA224 密码哈希 |
| [[core/pipeline/protocols/vless|VLESS]] | `vless()` | Session 层 | UUID |
| [[core/pipeline/protocols/shadowsocks|SS2022]] | `shadowsocks()` | 外层 ShadowTLS | PSK |

## 处理器统一签名

所有协议处理器遵循统一的函数签名：

```cpp
auto handler(session_context &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

参数说明：
- `ctx`: 会话上下文，包含入站传输、配置、内存资源
- `data`: 预读数据，协议检测时读取的初始数据

## 数据流向

```
Dispatch 层输出 (session_context + protocol_type)
       │
       ▼
Pipeline 层入口 (handler 函数)
       │
       ├── wrap_with_preview() 包装入站传输
       │
       ├── protocol::*::relay 创建中继器
       │
       ├── relay::handshake() 执行握手
       │
       ├── analysis::resolve() 解析目标地址
       │
       ├── 命令分发
       │     ├── TCP: dial() + tunnel()
       │     ├── UDP: make_datagram_router() + async_associate()
       │     ├── mux: bootstrap()
       │
       ▼
Channel 层 (传输层抽象)
```

## 相关文档

- [[core/pipeline/primitives|管道原语详解]]
- [[core/pipeline/protocols/http|HTTP 协议管道]]
- [[core/pipeline/protocols/socks5|SOCKS5 协议管道]]
- [[core/pipeline/protocols/trojan|Trojan 协议管道]]
- [[core/pipeline/protocols/vless|VLESS 协议管道]]
- [[core/pipeline/protocols/shadowsocks|SS2022 协议管道]]
## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| 统一函数签名 | 所有协议处理器必须遵循 `auto handler(session_context&, span<const byte>) -> awaitable<void>` | 编译错误或调度表无法统一 | `primitives.hpp` |
| header-only | 管道原语（shut_close/dial/tunnel/forward）全部 inline | 链接错误 | `connect/util.hpp`, `connect/tunnel/` |
| 预读数据不可丢 | `data` 参数包含 probe 阶段读取的初始字节，必须传递给协议处理器 | 首包数据丢失，握手失败 | 协议处理器签名 |
| 协程纯度 | 管道函数返回 `net::awaitable<void>`，禁止内部阻塞操作 | 死锁或 worker 线程饿死 | 协程约定 |
| PMR 内存 | `target` 结构体使用 `memory::string`，必须传入有效 `resource_pointer` | 内存分配器不匹配，运行时崩溃 | `protocol/common/target.hpp` |

## 故障场景

### 1. 协议探测错误导致处理器不匹配

**触发**: probe 阶段将连接误判为 HTTP，但实际是 Trojan over TLS。Dispatch 层将连接路由到 HTTP 处理器。

**表现**: HTTP 解析器收到 Trojan 二进制帧，解析失败返回 `bad_message` 或 `parse_error`，连接关闭。

**恢复**: 无法自动恢复。客户端重连时 probe 可能再次正确识别。TLS 内层协议的二次识别（recognition pipeline）可部分缓解。

### 2. 预读数据耗尽后读取阻塞

**触发**: `wrap_with_preview()` 包装入站传输后，协议握手需要的字节数超过预读缓冲区大小。

**表现**: 从 preview 层继续读取触发 `async_read_some`，正常情况下由底层 transport 填充。但如果对端发送延迟，协程挂起等待数据。

**恢复**: session 层的 `steady_timer` 超时机制保证不会永久挂起。超时后返回 `fault::timeout`。

### 3. 上游拨号失败后连接泄漏

**触发**: `dial()` 成功建立上游连接，但 `tunnel()` 双向转发启动前 session 被取消（如客户端 RST）。

**表现**: 上游连接已建立但未被转发，`shared_transmission` 的引用计数由 RAII 管理，session 析构时释放，不会实际泄漏。

**恢复**: RAII 保证资源释放。但上游连接的短暂存留会增加连接池/目标服务器的负担。

### 4. UDP 中继关联超时

**触发**: `async_associate()` 建立 UDP 会话后，客户端长时间未发送数据包。

**表现**: UDP 会话过期（`fault::udp_expired`），后续数据包被丢弃。客户端需重新发起 SOCKS5 UDP ASSOCIATE。

**恢复**: 协议处理器返回错误码，session 关闭对应会话。客户端行为决定是否重试。

### 5. Mux 目标检测误判

**触发**: 非多路复用目标的域名恰好以 `.mux.sing-box.arpa` 结尾。

**表现**: `is_mux()` 返回 true，连接被路由到 mux bootstrap 路径，尝试解析 mux 帧失败。

**恢复**: mux 处理失败后返回错误码，session 关闭连接。概率极低（保留域名后缀）。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| 协议类型枚举 | recognition → pipeline → dispatch | Probe 产出 `protocol_type`，dispatch 用于 switch 分发到对应处理器 |
| session_context 传递 | session → pipeline | session 构建上下文（传输层、配置、PMR 资源），pipeline 只消费不修改 |
| target 路由 | pipeline → resolve | 协议解析出的 `protocol::target` 传递给 `dial()` 进行上游路由 |
| 预读重放 | transport → pipeline | `transport::preview` 包装原始传输，透明回放 probe 阶段已读字节 |
| 错误码传播 | pipeline → session | 处理器通过 `co_return` 或异常退出，session 根据结果清理资源 |
| 连接拨号 | pipeline → connect | 处理器调用 `dial()` 建立上游连接，依赖 connect 层的路由/竞速/连接池 |
| 双向转发 | pipeline → connect/tunnel | 处理器调用 `tunnel()` 启动双向数据转发 |
| Mux 引导 | pipeline → multiplex | 处理器检测到 mux 目标后调用 `bootstrap()` 建立多路复用会话 |

## 变更敏感度

### 对外影响

| 变更类型 | 影响范围 | 破坏性 |
|----------|----------|--------|
| 处理器函数签名变更 | 所有协议处理器和 dispatch 分发逻辑需同步修改 | **高** |
| `protocol_type` 枚举变更（删除/重排） | switch 分发表、recognition 探测、日志格式全部受影响 | **高** |
| `target` 结构体字段变更 | 所有拨号和路由代码需适配 | **高** |
| 新增协议处理器 | 需在 dispatch switch 中添加 case，不影响已有处理器 | 低 |
| `is_mux()` 判定逻辑变更 | mux 引导行为可能改变 | 中 |

### 对内影响

| 变更类型 | 影响范围 | 风险 |
|----------|----------|------|
| `shut_close()` 逻辑变更 | 所有协议处理器的关闭路径受影响，可能引入资源泄漏 | 中 |
| `peel()` 解包逻辑变更 | native TLS 等依赖传输层解包的场景可能失效 | 中 |
| `wrap_with_preview()` 行为变更 | 所有协议处理器的首包读取行为改变 | **高** |
| PMR 内存资源传递方式变更 | `target` 等结构体的分配器初始化需同步修改 | 低 |
