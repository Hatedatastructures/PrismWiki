---
title: 六阶段流水线架构
created: 2026-05-17
layer: core
tags: [architecture, overview]
source: CLAUDE.md §架构概览
---

# 六阶段流水线架构

Prism 是高性能协程代理服务器，采用 C++23 纯协程架构和 PMR 内存池实现热路径零堆分配。

## 架构图

```
  Front Layer
  listener (监听) → balancer (负载均衡)
       │
       ▼
  Worker Layer
  worker (每线程一个 io_context) → launch (会话启动)
       │
       ▼
  Session Layer
  session (生命周期)
       │
       ▼
  Recognition Layer
  probe (外层探测) → clienthello (特征分析) → handshake (方案执行)
       │
       ▼
  Dispatch Layer (header-only)
  registry (处理器工厂) → handlers (Http/Socks5 等)
       │
       ▼
  Pipeline Layer
  protocols (http/socks5/trojan/vless/shadowsocks) → primitives (隧道)
       │
       ▼
  Resolve Layer
  router → recursor (DNS 门面) → cache/rules/resolver
       │
       ▼
  Multiplex Layer
  smux/yamux (多路复用协议) → duct (TCP 流) / parcel (UDP)
       │
       ▼
  Channel Layer
  connection (连接池) → eyeball (Happy Eyeballs)
  transport (reliable/encrypted/unreliable/transmission)
       │
       ▼
  Stealth Layer
  reality/shadowtls/restls (TLS 伪装) → schemes (方案执行)
```

## 各阶段职责

### Front Layer（前端层）

- **listener**: 监听端口，接受连接，计算亲和性哈希
- **balancer**: 选择 worker 分发连接

详见 [[agent/front/listener|监听器]]、[[agent/front/balancer|负载均衡器]]

### Worker Layer（工作线程层）

- **worker**: 每线程一个 io_context，处理连接
- **launch**: 会话启动，创建 session 对象

详见 [[agent/worker/worker|Worker]]、[[agent/worker/launch|会话启动]]

### Session Layer（会话层）

- **session**: 会话生命周期管理，持有 transport

详见 [[agent/session/session|会话生命周期]]

### Recognition Layer（识别层）

协议智能识别，三阶段流水线：

1. **probe**: 预读 24 字节，检测 HTTP/SOCKS5/TLS/SS2022
2. **clienthello**: 读取 TLS ClientHello，特征分析
3. **handshake**: 方案执行（Reality/ShadowTLS 等）

详见 [[recognition/recognition|协议识别入口]]、[[recognition/layered_pipeline|分层检测管道]]

### Dispatch Layer（分发层）

header-only 模块，处理器工厂：

- **registry**: 处理器注册表
- **handlers**: Http/Socks5/Trojan/Vless/Shadowsocks 处理器

详见 [[agent/dispatch/table|处理器分发表]]

### Pipeline Layer（管道层）

协议处理原语：

- **protocols**: 各协议处理器实现
- **primitives**: 隧道原语（透明转发）

详见 [[pipeline/overview|管道层总览]]

### Resolve Layer（解析层）

DNS 解析管道：

- **router**: 连接池管理
- **recursor**: DNS 门面，七阶段查询管道

详见 [[resolve/router|DNS 路由器]]、[[resolve/dns/dns|DNS 解析入口]]

### Multiplex Layer（多路复用层）

连接复用：

- **smux/yamux**: 多路复用协议
- **duct**: TCP 流管道
- **parcel**: UDP 数据报管道

详见 [[multiplex/overview|多路复用总览]]

### Channel Layer（通道层）

传输层抽象：

- **connection**: 连接池
- **eyeball**: Happy Eyeballs（IPv4/IPv6 双栈）
- **transport**: reliable(TCP)/encrypted(TLS)/unreliable(UDP)

详见 [[channel/overview|通道层总览]]、[[channel/connection/pool|连接池]]

### Stealth Layer（伪装层）

TLS 指纹伪装：

- **reality**: Reality 方案
- **shadowtls**: ShadowTLS 方案
- **restls**: Restls 方案
- **schemes**: 方案执行器

详见 [[stealth/overview|伪装层总览]]

## 协程架构

所有异步操作返回 `net::awaitable<T>`：

- 使用 `co_await` 执行顺序异步操作
- 使用 `net::co_spawn` 启动独立协程
- 协程中禁止阻塞操作

详见 [[dev/coding/coroutine|协程约定]]

## PMR 内存策略

热路径零堆分配：

- `memory::string` — 使用全局池的 std::pmr::string
- `memory::vector<T>` — 使用帧竞技场的 std::pmr::vector<T>

详见 [[dev/coding/pmr|PMR 使用规范]]