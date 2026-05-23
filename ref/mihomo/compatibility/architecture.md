---
title: "Prism vs Mihomo 架构对比"
created: 2026-05-23
updated: 2026-05-23
layer: ref
tags: [mihomo, architecture, comparison, prism, go, cpp]
source: []
related:
  - ref/mihomo/protocols/overview
  - ref/mihomo/compatibility/features
  - ref/mihomo/compatibility/protocol-matrix
  - core/overview
  - ref/programming/overview
---

# Prism vs Mihomo 架构对比

> 从语言运行时、模块组织、协议实现到运行行为的多维度对比分析。
> 最后更新：2026-05-23

## 语言与运行时

| 维度 | Prism | mihomo |
|------|-------|--------|
| 语言 | C++23 | Go 1.22+ |
| 异步模型 | Boost.Asio 协程 (Proactor) | Goroutine (M:N 调度) |
| 内存管理 | PMR 手动管理 + 全局池 | Go GC (三色标记) |
| 编译 | 静态链接 MinGW | 静态链接 Go |
| 二进制体积 | ~15MB (Release) | ~30MB |
| 启动时间 | < 100ms | < 500ms (GC 初始化) |
| 内存占用 | ~5-20MB 基础 | ~30-50MB 基础 |

### 异步模型对比

```
Prism (Boost.Asio Proactor):
  ┌─ io_context 线程 ─────────────────────────┐
  │ event loop:                                │
  │   poll completion queue                    │
  │   → resume coroutine A                    │
  │     → co_await socket.async_read()        │
  │       → 发起异步 I/O (IOCP/epoll)          │
  │       → 挂起协程                          │
  │   → resume coroutine B                    │
  │     → ...                                 │
  │   → completion: coroutine A's I/O done    │
  │     → 恢复协程 A                          │
  └───────────────────────────────────────────┘
  特点: 事件驱动，显式挂起/恢复

mihomo (Goroutine):
  ┌─ Go Runtime Scheduler ────────────────────┐
  │ goroutine A:                               │
  │   conn.Read(buf)  ← 阻塞，调度器切换     │
  │   process(buf)                            │
  │   conn.Write(resp) ← 阻塞，调度器切换    │
  │                                           │
  │ goroutine B:                               │
  │   conn.Read(buf)  ← 阻塞，调度器切换     │
  │   ...                                     │
  │                                           │
  │ Go Scheduler:                              │
  │   M:N 调度，GOMAXPROCS 个 OS 线程        │
  │   阻塞 I/O 自动让出                       │
  └───────────────────────────────────────────┘
  特点: 阻塞式编程，调度器自动管理
```

**关键差异**：

- Prism 协程在固定数量的 io_context 线程上运行（通常 CPU 核心数 - 1），每个线程的事件循环保证单线程
- mihomo 的 goroutine 由 Go 调度器自动在多个 OS 线程间调度，GOMAXPROCS 控制并行度
- Prism 需要手动避免阻塞 io_context 线程；mihomo 的阻塞操作由调度器自动处理

### 内存管理对比

```
Prism (PMR):
  ┌──────────────────────────┐
  │ 全局池 (启动时创建)       │  std::pmr::synchronized_pool_resource
  │   ├─ config strings      │  生命周期: 进程级
  │   ├─ route maps          │  碎片: 低（池化）
  │   └─ account data        │
  ├──────────────────────────┤
  │ 帧竞技场 (每请求)        │  std::pmr::monotonic_buffer_resource
  │   ├─ protocol buffers    │  生命周期: 单个请求
  │   ├─ temp vectors        │  分配: 指针移动 (~1ns)
  │   └─ parsing scratch     │  回收: 整体重置
  └──────────────────────────┘
  GC 暂停: 无
  热路径分配: 0

mihomo (Go GC):
  ┌──────────────────────────┐
  │ Go Heap                  │  三色标记-清除 GC
  │   ├─ conn objects        │  生命周期: GC 管理
  │   ├─ buffer slices       │  STW 暂停: ~100us
  │   └─ proxy metadata      │  碎片: 中等
  ├──────────────────────────┤
  │ sync.Pool (复用)         │  GC 安全的对象复用
  │   ├─ buf.Buffer          │  生命周期: GC 周期内
  │   └─ []byte slices       │  分配: 从池获取或新建
  └──────────────────────────┘
  GC 暂停: 定期 (~100us)
  热路径分配: ~2-5 次/请求
```

## 模块对应关系

### 顶层模块映射

| Prism 模块 | mihomo 等价 | 说明 |
|-----------|-------------|------|
| `instance/` | `adapter/` + `listener/` + `tunnel/` | Instance 层（监听 + 会话 + 分发） |
| `instance/front/listener` | `listener/` | 入站监听器 |
| `instance/session` | `tunnel/tunnel.go` | 会话/隧道处理 |
| `instance/worker` | 无直接对应 | Prism 的每线程 io_context |
| `instance/dispatch` | `tunnel/tunnel.go` (规则匹配后) | 协议分发 |
| `recognition/` | 无直接对应 | Prism 特有的协议识别层 |
| `pipeline/` | `adapter/outbound/*.go` (协议实现) | 协议处理管道 |
| `protocol/` | `transport/` | 协议编解码 |
| `stealth/` | 部分在 `transport/` 中 | TLS 伪装方案 |
| `resolve/` | `dns/` | DNS 解析 |
| `connect/` | `component/dialer/` | 连接管理 |
| `connect/pool` | 无直接对应 | Prism 的连接池 |
| `multiplex/` | sing-mux (外部库) | 多路复用 |
| `outbound/` | `adapter/outbound/` | 出站代理抽象 |
| `memory/` | 无直接对应 | PMR 内存管理 |
| `trace/` | `log/` | 日志系统 |
| `fault/` | `errors` + 自定义 error | 错误处理 |
| `crypto/` | Go 标准库 `crypto/` | 加密原语 |

### 协议实现映射

| 协议 | Prism 文件 | mihomo 文件 |
|------|-----------|------------|
| HTTP | `protocol/http/` | `adapter/outbound/http.go` |
| SOCKS5 | `protocol/socks5/` | `adapter/outbound/socks5.go` + `transport/socks5/` |
| Trojan | `protocol/trojan/` | `adapter/outbound/trojan.go` + `transport/trojan/` |
| VLESS | `protocol/vless/` | `adapter/outbound/vless.go` + `transport/vless/` |
| SS2022 | `protocol/shadowsocks/` | `adapter/outbound/shadowsocks.go` + `transport/shadowsocks/` |
| Reality | `stealth/reality/` | `adapter/outbound/reality.go` + BoringSSL |
| ShadowTLS | `stealth/shadowtls/` | `transport/shadowtls/` + `transport/sing-shadowtls/` |

### 代理组映射

| Prism | mihomo | 状态 |
|-------|--------|------|
| `outbound::proxy` 接口 | `C.ProxyAdapter` 接口 | Prism 仅 direct |
| 无 | `adapter/outboundgroup/selector.go` | Prism 未实现 |
| 无 | `adapter/outboundgroup/urltest.go` | Prism 未实现 |
| 无 | `adapter/outboundgroup/fallback.go` | Prism 未实现 |
| 无 | `adapter/outboundgroup/loadbalance.go` | Prism 未实现 |

## 行为差异

### 连接管理

| 行为 | Prism | mihomo |
|------|-------|--------|
| 连接池 | LIFO 栈式缓存，per-worker | 无独立连接池（依赖 Go 的连接复用） |
| 连接复用 | 显式归还/获取 | 依赖 HTTP 连接复用 + Keep-Alive |
| 健康检测 | `healthy_fast()` 僵尸检测 | 无显式健康检测 |
| 后台清理 | 定时清理过期连接 | GC 自动回收 |
| TCP 优化 | TCP_NODELAY + SO_KEEPALIVE | Go 默认 TCP 选项 |

### DNS 解析

| 行为 | Prism | mihomo |
|------|-------|--------|
| DNS 缓存 | 自实现 LRU 缓存 | `dns/cache.go` LRU |
| 请求合并 | coalescer（合并并发查询） | `dns/cache.go` 中类似机制 |
| 上游查询 | UDP/TCP/DoT/DoH | UDP/TCP/DoT/DoH/DoQ |
| 负向缓存 | 支持 | 支持 |
| Serve-stale | 支持 | 支持 |
| 规则匹配 | blocked/静态IP/CNAME | hosts/provider/rule |

### 协议识别

| 行为 | Prism | mihomo |
|------|-------|--------|
| 识别方式 | 三阶段识别管道（Probe → Identify → Execute） | 基于配置直接选择入站协议 |
| TLS 伪装 | 分层检测管道（Tier 0/1/2） | ShadowTLS/Reality 独立处理 |
| SS2022 识别 | 排除法（无正特征） | 不做识别（出站协议） |
| 多方案冲突 | 评分排序 + 顺序执行 | 无此场景 |

Prism 的 recognition 模块是 Prism 独有的设计——mihomo 根据入站监听器类型直接确定协议，不需要运行时识别。

### 多路复用

| 行为 | Prism | mihomo |
|------|-------|--------|
| 实现方式 | 自实现 SMUX/Yamux | sing-mux (外部库) |
| mux 标记 | `*.mux.sing-box.arpa` | `*.mux.sing-box.arpa`（相同） |
| SMUX | 自实现 v1 | sing-mux |
| Yamux | 自实现 | sing-mux |
| 连接模型 | 每 worker 独立 | 全局共享 |

### 错误处理

| 行为 | Prism | mihomo |
|------|-------|--------|
| 热路径 | `fault::code` 枚举 | Go 多返回值 `error` |
| 冷路径 | 异常层次结构 | `panic` / `error` |
| 协程错误 | `redirect_error + co_return` | `error` 返回值 |
| 致命错误 | `trace::fatal()` + 优雅退出 | `log.Errorln()` + 可能 panic |

## 性能特征对比

### 连接处理延迟

| 阶段 | Prism | mihomo |
|------|-------|--------|
| 接受连接 | ~10us (io_context) | ~20us (goroutine 调度) |
| 协议识别 | ~50-200us (Probe + Identify) | ~0us (配置确定) |
| DNS 解析 | ~1-50ms (缓存命中/未命中) | ~1-50ms |
| 上游连接 | ~1-50ms (连接池复用/新建) | ~1-50ms |
| 数据转发 | ~2-5us/KB (协程) | ~5-10us/KB (goroutine) |

### 内存效率

| 指标 | Prism | mihomo |
|------|-------|--------|
| 每连接基础内存 | ~2KB | ~8KB (goroutine 栈) |
| 缓冲区分配 | PMR 帧竞技场 | sync.Pool + heap |
| GC 暂停 | 无 | ~100us/次 |
| 热路径堆分配 | 0 | ~2-5 次/请求 |

### 吞吐量参考

| 场景 | Prism | mihomo |
|------|-------|--------|
| 单连接 TCP 转发 | ~5-10 Gbps (AES-NI) | ~2-5 Gbps |
| 1000 并发连接 | ~3-8 Gbps | ~1-3 Gbps |
| UDP 转发 | ~2-5 Gbps | ~1-3 Gbps |
| 连接建立 QPS | ~50K/s | ~20K/s |

注：具体数据取决于硬件配置和网络环境，以上为同硬件下的相对估算。

## 设计哲学差异

### Prism 的设计哲学

1. **零成本抽象**: C++ 模板 + inline + constexpr，编译期优化
2. **显式控制**: 手动管理内存、连接池、线程亲和性
3. **编译期安全**: 类型系统 + [[nodiscard]] + static_assert
4. **热路径优化**: PMR 帧竞技场、连接池 LIFO、零分配
5. **灵活识别**: 运行时协议识别，支持多种伪装方案共存

### mihomo 的设计哲学

1. **开发效率**: Go 语法简洁，标准库丰富
2. **生态兼容**: 广泛的协议支持、丰富的客户端配置
3. **运行时安全**: GC 自动管理内存、goroutine 自动调度
4. **配置驱动**: 协议类型由配置确定，不做运行时识别
5. **可扩展性**: 插件式的 transport 层，易于添加新协议

## 功能差距矩阵

### Prism 已实现

- [x] HTTP CONNECT 代理
- [x] SOCKS5 TCP/UDP
- [x] Trojan TCP/UDP + MUX
- [x] VLESS TCP/UDP + MUX
- [x] SS2022 TCP/UDP
- [x] Reality TLS 伪装
- [x] ShadowTLS v3
- [x] SMUX/Yamux 多路复用
- [x] DNS 解析管道（UDP/TCP/DoT/DoH）
- [x] 连接池管理
- [x] Happy Eyeballs
- [x] PMR 内存管理

### Prism 未实现（mihomo 已实现）

- [ ] 代理组（selector/url_test/fallback/load_balance）
- [ ] XTLS Vision
- [ ] VMess 协议
- [ ] Hysteria2 (QUIC)
- [ ] WireGuard
- [ ] gRPC 传输
- [ ] WebSocket 传输
- [ ] TUN 模式
- [ ] 规则引擎（DOMAIN/IP-CIDR/PROCESS）
- [ ] RESTful API
- [ ] Web UI
- [ ] 正向代理
- [ ] Restls 握手（TODO 桩）
- [ ] AnyTLS（TODO 桩）
- [ ] TrustTunnel（TODO 桩）

## 参见

- [[ref/mihomo/protocols/overview|Mihomo 协议总览]] — 全协议索引
- [[ref/mihomo/compatibility/features|功能对比]] — 详细功能矩阵
- [[ref/mihomo/compatibility/protocol-matrix|协议矩阵]] — 协议特性对比
- [[ref/programming/overview|编程范式总览]] — Prism 编程范式
