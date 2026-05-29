---
title: "channel — 通道阶段概述"
layer: core
source: include/prism/connect/pool/health.hpp
  - include/prism/connect/
  - include/prism/transport/
module: "channel"
type: overview
tags: [channel, transport, transmission, connection-pool, TLS]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/connect/pool/health
  - core/transport/transmission
  - core/connect/pool/pool
  - core/transport/adapter/connector
  - core/connect/dial/racer
---

# channel — 通道阶段概述

> 源码: `include/prism/connect/` + `include/prism/transport/`
> 模块: [[channel|channel]]
> 定位: 请求处理流程的第五阶段

## 模块职责

通道阶段（Channel Layer）是 Prism 请求处理流程的第五阶段，位于 Pipeline 层与 Outbound 层之间。该阶段负责建立和管理与上游服务器的网络连接，提供统一的传输层抽象，实现 TCP 连接复用和 TLS 加密封装。

核心职责：

| 职责 | 说明 | 关键组件 |
|------|------|----------|
| 传输层抽象 | 统一 TCP/UDP/TLS 的读写接口 | [[core/transport/transmission|transmission]] |
| TCP 可靠传输 | 封装 TCP socket，支持连接池复用 | [[core/transport/reliable|reliable]] |
| TLS 加密传输 | 封装 TLS 流，提供加密读写 | [[core/transport/encrypted|encrypted]] |
| UDP 数据报传输 | 封装 UDP socket，模拟连接式操作 | [[core/transport/unreliable|unreliable]] |
| 预读回滚传输 | 捕获读取数据，支持 rewind 重试 | [[core/transport/snapshot|snapshot]] |
| 连接池管理 | LIFO 栈式缓存，健康检测，后台清理 | [[core/connect/pool/pool|pool]] |
| Happy Eyeballs | 多地址并发竞速连接 | [[core/connect/dial/racer|racer]] |
| Socket 适配器 | 适配 Boost.Asio 流概念 | [[core/transport/adapter/connector|connector]] |
| 健康检测 | Socket 状态检测，防止复用僵尸连接 | [[core/connect/pool/health|health]] |

## 源码结构

```
channel/
├── health.hpp               # Socket 健康检测
│
├── transport/               # 传输层实现
│   ├── transmission.hpp     # 抽象接口基类
│   ├── reliable.hpp         # TCP 可靠传输
│   ├── encrypted.hpp        # TLS 加密传输
│   ├── unreliable.hpp       # UDP 数据报传输
│   └── snapshot.hpp         # 预读回滚传输
│
├── connection/              # 连接管理
│   └── pool.hpp             # TCP 连接池
│
├── eyeball/                 # 并发连接
│   └── racer.hpp            # Happy Eyeballs 竞速器
│
└── adapter/                 # 适配器
    └── connector.hpp        # Boost.Asio 适配器
```

## 调用链

### 上游调用（入）

通道阶段接收以下上游模块的请求：

| 上游模块 | 调用入口 | 说明 |
|----------|----------|------|
| Pipeline 层 | `transmission::async_read_some()` | 协议读写使用传输层接口 |
| Resolve 层 | `address_racer::race()` | 通过 Router 获取端点列表后竞速连接 |
| Multiplex 层 | `transmission::async_write_scatter()` | 多路复用帧读写 |
| Stealth 层 | `snapshot::rewind()` | TLS 伪装方案依次尝试 |

### 下游调用（出）

通道阶段调用以下下游模块：

| 下游模块 | 调用出口 | 说明 |
|----------|----------|------|
| Boost.Asio | `tcp::socket::async_connect()` | TCP 连接建立 |
| Boost.Asio | `ssl::stream::async_handshake()` | TLS 握手 |
| [[core/connect/pool/health|health]] | `healthy_fast()` | 连接池复用前健康检测 |

### 内部调用链

```
router::async_forward
  → address_racer::race(endpoints)
    → connection_pool::async_acquire(ep)
      ├─→ 缓存命中 → healthy_fast() → pooled_connection
      └─→ 缓存未命中 → tcp::socket::async_connect
    → pooled_connection 析构
      → connection_pool::recycle()
        → healthy_fast() → 入栈或销毁
```

## 模块依赖

### 依赖关系图

```
channel/
├── health.hpp           → 无内部依赖（基础模块）
│
├── transport/
│   ├── transmission     ← 抽象基类
│   ├── reliable         → 依赖 pool (pooled_connection)
│   ├── encrypted        → 依赖 connector
│   ├── unreliable       → 无内部依赖
│   └── snapshot         → 依赖 transmission (inner)
│
├── connection/pool      → 依赖 health
│
├── eyeball/racer        → 依赖 pool
│
└── adapter/connector    → 依赖 transmission
```

## 关键设计

### transmission 抽象接口

[[core/transport/transmission|transmission]] 是所有传输层的抽象基类，采用纯协程设计：

- 异步操作返回 `net::awaitable`
- 错误通过 `std::error_code&` 参数返回
- 支持 Scatter-gather 写入优化
- 兼容 Boost.Asio 执行器概念

### 连接池 LIFO 策略

[[core/connect/pool/pool|connection_pool]] 采用 LIFO 栈式缓存：

- 最近使用的连接在栈顶，优先复用
- 降低过期风险
- 线程隔离，无锁竞争
- 后台定时清理过期连接

### Happy Eyeballs 算法

[[core/connect/dial/racer|address_racer]] 实现 RFC 8305 并发连接：

- 第一个端点立即尝试
- 后续端点延迟 250ms 依次尝试
- 第一个成功的连接 wins
- 最大并发数限制为 6

## 参见

- [[core/connect/pool/health|health]] — Socket 健康检测
- [[core/transport/transmission|transmission]] — 传输层抽象接口
- [[core/transport/reliable|reliable]] — TCP 可靠传输
- [[core/transport/encrypted|encrypted]] — TLS 加密传输
- [[core/transport/unreliable|unreliable]] — UDP 数据报传输
- [[core/transport/snapshot|snapshot]] — 预读回滚传输
- [[core/connect/pool/pool|pool]] — TCP 连接池
- [[core/connect/dial/racer|racer]] — Happy Eyeballs 竞速器
- [[core/transport/adapter/connector|connector]] — Boost.Asio 适配器
## 设计决策（WHY）

### 为什么连接池使用 LIFO 栈式缓存

LIFO 策略保证最近使用的连接在栈顶优先复用，降低连接过期风险。后进先出意味着热端点的高频连接总被优先取回，冷端点连接自然沉底被后台清理淘汰。相比 FIFO 或随机选取，LIFO 在连接存活窗口内最大化复用率。

### 为什么采用 Happy Eyeballs RFC 8305 算法

DNS 解析通常返回多个 IP 地址（IPv4 + IPv6）。串行尝试导致首地址不可达时延迟叠加。RFC 8305 的 250ms 交错启动策略在保持 IPv6 优先的同时，快速回退到可用地址。竞速器内部通过共享 race_context 的 winner 标志实现"第一个成功 wins"语义，无需互斥锁。

### 为什么 transmission 采用抽象基类而非 concept

transmission 是所有传输层的统一接口，被 protocol 和 stealth 模块大量使用。抽象基类允许在运行时擦除具体传输类型（TCP/TLS/UDP），支持将加密传输无缝替换为明文传输。virtual 函数开销在此场景可忽略（IO 密集），换来的是跨层多态灵活性。

### 为什么连接池设计为线程局部（无锁）

每个 worker 线程持有独立的 io_context 和连接池。线程隔离消除了连接获取/归还路径上的互斥竞争，LIFO 栈操作在单线程上是 O(1)。这要求连接只能在创建它的线程中归还，pooled_connection 的 RAII 语义自然保证这一点。

### 为什么 pooled_connection 使用 RAII 包装器

连接的获取和归还必须配对。RAII 包装器通过析构函数自动调用 recycle()，即使协程异常退出也能正确归还连接。release() 方法支持转移 socket 所有权（如交给 tunnel），此时析构不会归还。移动语义确保连接在协程间安全传递。

## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| 线程隔离 | connection_pool 单线程使用 | 数据竞争、UB | 无锁设计前提 |
| io_context 生命周期 | pool 析构前 io_context 必须有效 | 定时器回调崩溃 | pool.hpp 注释 |
| RAII 归还 | pooled_connection 析构触发 recycle | 连接泄漏 | 资源管理 |
| 健康检测 | 复用前必须 healthy_fast() | 使用已关闭连接导致 EPIPE | 连接可靠性 |
| 协程纯度 | async_acquire 返回 awaitable | 阻塞 worker 线程 | 协程约定 |
| 容量限制 | 每端点缓存数由 config 控制 | 内存持续增长 | pool config |
| racer 竞速安全 | racer 是局部变量，子协程捕获 pool 引用 | racer 析构后协程访问悬挂引用 | racer.hpp 注释 |

## 故障场景

### 1. 连接池返回僵尸连接

对端已发送 FIN 但连接仍在池中。healthy_fast() 通过非阻塞 recv(MSG_PEEK) 检测到此状态，将连接标记为不健康并关闭。若检测遗漏（极端竞态），协议层首次读写会收到 EOF/error，触发正常连接关闭和重试。

### 2. 所有 racer 端点不可达

address_racer::race() 尝试所有候选端点后无 winner，返回空 pooled_connection。调用方（router）检测到空连接后返回 fault::code::bad_gateway，最终传递到协议层返回连接失败响应。

### 3. 连接池容量耗尽

当某端点的缓存栈达到容量上限时，recycle() 直接关闭归还的连接而非入栈。统计计数器 total_evictions 递增。不会导致崩溃，但复用率下降，后续请求需新建连接。

### 4. 后台清理定时器未启动

若 start() 未在 io_context.run() 前调用，过期连接不会被自动清理。连接池逐渐积累失效连接，占用文件描述符。不影响正确性（复用前仍有健康检测），但资源浪费持续增长。

### 5. snapshot rewind 后原始传输状态不一致

snapshot（预读回滚传输）在 rewind 时恢复读取位置，但底层传输可能已经消费了更多数据。若 rewind 后的数据与预期不符，伪装方案切换会失败。需要确保 snapshot 捕获的数据量不超过内部缓冲区容量。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| transmission 接口 | channel -> protocol/stealth | 所有传输层实现此抽象基类，协议和伪装模块通过接口读写 |
| connection_pool 引用 | channel <- connect::router | router 持有 pool 引用，通过 pool 获取连接 |
| pooled_connection RAII | channel -> connect::tunnel | tunnel 接管连接后通过 release() 获取裸 socket |
| health 检测契约 | pool -> health | pool 在 acquire 和 recycle 时调用 healthy_fast() |
| address_racer 竞速 | router -> racer -> pool | router 提供端点列表，racer 通过 pool 竞速连接 |
| snapshot rewind | stealth -> snapshot | 伪装方案依次尝试，失败时 rewind 恢复到初始状态 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 说明 |
|------|----------|------|
| transmission 接口变更 | protocol 所有 relay、stealth 所有 scheme | 核心抽象，广泛依赖 |
| pooled_connection API 变更 | router、racer、forward | 连接生命周期管理依赖 |
| healthy_fast() 行为变更 | 连接池复用质量 | 过严导致复用率下降，过松导致 EPIPE |

### 对内影响

| 变更 | 影响范围 | 说明 |
|------|----------|------|
| LIFO 栈实现替换 | pool 内部、stats 计算 | 缓存策略变更影响复用行为 |
| racer 延迟参数调整 | Happy Eyeballs 行为 | 250ms 延迟为 RFC 建议值，偏离需性能验证 |
| 健康检测阈值调整 | 僵尸连接检出率 | 影响 pool 质量和连接建立延迟 |
| config 参数变更 | pool 容量、超时、清理间隔 | 运维调优，不影响代码逻辑 |
