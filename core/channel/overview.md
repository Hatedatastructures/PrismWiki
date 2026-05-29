---
title: "channel — 通道阶段概述"
layer: core
source:
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