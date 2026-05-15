---
title: Agent 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [agent, core, overview, design]
related: [[recognition], [pipeline], [channel/overview], [resolve/overview], [outbound], [stealth], [loader], [memory/overview]]
---

# Agent 模块

Agent 模块是 Prism 的**核心会话链路模块**，负责从前端监听到会话分发的完整连接生命周期。它是其他所有模块的编排者：监听端口接受连接，通过负载均衡分发到工作线程，创建会话后调用 recognition 识别协议，再通过 dispatch 分派到 pipeline 处理。

## 概述

### 核心职责

- **前端监听**：绑定端口，接受入站 TCP 连接，支持反压机制
- **负载均衡**：基于加权评分选择最优工作线程，支持过载检测
- **会话管理**：管理单个连接的完整生命周期（识别 → 分发 → 转发）
- **协议分发**：编译期函数表替代虚函数，零开销分发到对应处理管道
- **账户管理**：基于凭证的用户认证和连接数限制，无锁读取 + 写时复制

### 模块结构

```
include/prism/agent/
├── config.hpp                # Agent 配置类型定义
├── context.hpp               # 运行时上下文（server/worker/session 三级）
├── account/
│   ├── entry.hpp             # 账户条目状态 + 租约 RAII
│   └── directory.hpp         # 账户目录（无锁读取 + COW 更新）
├── dispatch/
│   └── table.hpp             # 编译期协议处理函数表
├── front/
│   ├── listener.hpp          # 前端监听器
│   └── balancer.hpp          # 负载均衡器
├── session/
│   └── session.hpp           # 会话生命周期管理
└── worker/
    ├── worker.hpp            # 工作线程核心
    ├── launch.hpp            # 会话启动与连接分发
    ├── stats.hpp             # 负载统计（EMA 平滑延迟）
    └── tls.hpp               # TLS 上下文初始化
```

### 架构概览

```
listener（监听）→ balancer（负载均衡）
     │
     ▼
worker（per-thread io_context）→ launch（会话启动）
     │
     ▼
session（生命周期管理）
     │
     ▼
recognition::recognize() → dispatch::table[type]
     │
     ▼
pipeline::http/socks5/trojan/... → primitives::tunnel()
```

### 数据流

```
1. listener::accept_loop() 接受连接
2. balancer::select(affinity) 选择 worker
3. worker::dispatch_socket() 跨线程投递
4. launch::start() 创建会话
5. session::start() → diversion()
6. recognition::recognize() 识别协议
7. dispatch::dispatch() 分派到 handler
8. pipeline handler → primitives::tunnel()
9. session::close() 关闭释放
```

## 详细设计

### listener（前端监听器）

**文件**: `front/listener.hpp`

```cpp
class listener {
public:
    explicit listener(const psm::config &cfg, balancer &dispatcher);
    void listen();  // 启动监听，阻塞运行 io_context
private:
    static auto make_affinity(const tcp::endpoint &endpoint) noexcept -> std::uint64_t;
    auto accept_loop() -> net::awaitable<void>;
    net::io_context ioc_;                          // 独立 IO 上下文
    tcp::acceptor acceptor_;
    balancer &dispatcher_;
    std::uint32_t buffer_size_;
    std::chrono::milliseconds backpressure_delay_;
};
```

独立 io_context 运行，与 worker 隔离。accept_loop 协程持续接受连接，根据客户端端点计算亲和性哈希，调用 balancer 选择目标 worker 并分发。当 balancer 返回反压标志时，暂停指定延迟后再继续接受。

### balancer（负载均衡器）

**文件**: `front/balancer.hpp`

```cpp
struct worker_load_snapshot {
    std::uint32_t active_sessions{0};
    std::uint32_t pending_handoffs{0};
    std::uint64_t event_loop_lag_us{0};
};

class balancer {
public:
    struct select_result { std::size_t worker_index{0}; bool overflowed{false}; bool backpressure{false}; };
    explicit balancer(memory::vector<worker_binding> bindings, const distribute_config &config = {});
    auto select(std::uint64_t affinity_value) noexcept -> select_result;
    void dispatch(std::size_t worker_index, tcp::socket socket) const;
};
```

选择算法：根据亲和性值计算候选 → 获取实时负载快照 → 加权评分（session 0.6 + pending 0.1 + lag 0.3）→ 选择评分最低的健康 worker。过载检测采用滞后机制避免抖动。

### session（会话管理器）

**文件**: `session/session.hpp`

```cpp
class session : public std::enable_shared_from_this<session> {
public:
    enum class state : std::uint8_t { active, closing, closed };
    explicit session(session_params params);
    void start();
    void close();    // 幂等
    [[nodiscard]] std::uint64_t id() const noexcept;
private:
    auto diversion() -> net::awaitable<void>;  // 协议分流：recognize → dispatch
    void release_resources() noexcept;
    std::uint64_t id_;
    memory::frame_arena frame_arena_;
    state state_{state::active};
    session_context ctx_;
};
```

生命周期管理采用"先停、再收"模型：close() 只标记关闭状态、取消底层连接，不立即 reset 传输对象；资源释放在主处理协程退出后或析构时统一进行。diversion() 是核心流程：调用 recognition::recognize() 识别协议，然后通过 dispatch::dispatch() 分派到对应 handler。

### worker（工作线程）

**文件**: `worker/worker.hpp`

```cpp
class worker {
public:
    explicit worker(const psm::config &cfg, std::shared_ptr<account::directory> account_store);
    void run();                                 // 启动事件循环（阻塞）
    void dispatch_socket(tcp::socket socket);   // 跨线程分发（线程安全）
    auto load_snapshot() const noexcept -> front::worker_load_snapshot;
private:
    net::io_context ioc_;
    connection_pool pool_;
    resolve::router router_;
    std::shared_ptr<ssl::context> ssl_ctx_;
    stats::state metrics_;
    server_context server_ctx_;
    worker_context worker_ctx_;
};
```

每个 worker 拥有独立的 io_context、连接池、路由器，实现线程间资源隔离。dispatch_socket 和 load_snapshot 是线程安全的，其余方法必须在 worker 线程内调用。

### dispatch::table（协议处理函数表）

**文件**: `dispatch/table.hpp`

```cpp
using handler_func = net::awaitable<void>(session_context &, std::span<const std::byte>);

inline constexpr std::array<handler_func *, static_cast<std::size_t>(protocol_type::tls) + 1>
    handler_table{
        handle_unknown,        // unknown: 原始 TCP 透传
        pipeline::http,        // http
        pipeline::socks5,      // socks5
        pipeline::trojan,      // trojan
        pipeline::vless,       // vless
        pipeline::shadowsocks, // shadowsocks
        handle_unknown,        // tls: 由 stage chain 处理
    };
```

编译期常量数组，零虚函数、零动态分配。索引对应 protocol_type 枚举值，越界时 fallback 到 unknown handler。

### context（三级上下文）

**文件**: `context.hpp`

```cpp
struct server_context {
    std::atomic<std::shared_ptr<const psm::config>> cfg;    // 可原子交换（热加载）
    std::shared_ptr<ssl::context> ssl_ctx;
    std::shared_ptr<account::directory> account_store;
};

struct worker_context {
    net::io_context &io_context;
    resolve::router &router;
    memory::resource_pointer memory_pool;
    outbound::proxy *outbound{nullptr};
};

struct session_context {
    std::uint64_t session_id{0};
    const server_context &server;
    worker_context &worker;
    memory::frame_arena &frame_arena;
    std::uint32_t buffer_size;
    shared_transmission inbound;
    shared_transmission outbound;
    account::lease account_lease;
};
```

### account（账户管理）

**文件**: `account/entry.hpp`, `account/directory.hpp`

```cpp
struct entry {
    std::uint32_t max_connections{0};
    std::atomic_uint64_t uplink_bytes{0};
    std::atomic_uint64_t downlink_bytes{0};
    std::atomic_uint32_t active_connections{0};
};

class directory {
public:
    void upsert(std::string_view credential, std::uint32_t max_connections = 0);
    auto find(std::string_view credential) const noexcept -> std::shared_ptr<entry>;
};

auto try_acquire(const directory &accounts, std::string_view credential) noexcept -> lease;
```

directory 使用原子共享指针 + 写时复制实现无锁读取。insert() 支持多协议凭证共享同一个 entry（如密码 + UUID 共享连接配额）。lease 为 RAII 类型，析构时自动递减 active_connections。

### launch（会话启动）与 stats（负载统计）

**文件**: `worker/launch.hpp`, `worker/stats.hpp`

dispatch() 实现跨线程连接分发：先递增待处理计数 → post 到目标 io_context → 在 worker 线程中 migrate_executor + prime + start。stats::state 的 observe 协程每 250ms 采样事件循环延迟，使用 EMA 平滑算法过滤抖动，预热阶段采集 16 个样本估算基线。

### config（配置结构）

**文件**: `config.hpp`

```cpp
struct config {
    limit limits;
    endpoint positive;                                 // 正向代理端点
    endpoint addressable;                              // 监听端点
    certificate cert;
    authentication auth;                               // users 列表（password + uuid）
    memory::string camouflage;                         // 伪装路径
    memory::map<memory::string, endpoint> reverse_map; // 反向代理路由表
};
```

authentication::user 结构支持同时配置 password（Trojan/HTTP/SOCKS5）和 uuid（VLESS），两种凭证共享同一个 entry，共享连接数配额。

## 模块依赖

| 方向 | 模块 | 用途 |
|------|------|------|
| 依赖 | `memory` | PMR 容器和内存池 |
| 依赖 | `channel` | 传输层抽象、连接池 |
| 依赖 | `resolve` | DNS 路由器 |
| 依赖 | `protocol` | `protocol_type`、`analysis` |
| 依赖 | `pipeline` | 协议处理函数（http/socks5/trojan/vless/shadowsocks） |
| 依赖 | `recognition` | `recognize()` 协议识别 |
| 依赖 | `outbound` | 出站代理抽象 |
| 依赖 | `stealth` | 伪装配置 |
| 被依赖 | `main.cpp` | 启动入口 |
| 被依赖 | `recognition` | 使用 `session_context` |
| 被依赖 | `pipeline` | 使用 `session_context` |

- [[docs/agent/troubleshooting]] — 故障排查
## 相关链接

- [[docs/agent/architecture]] — 架构设计
- [[docs/agent/configuration]] — 配置详解
- [[docs/agent/deployment]] — 部署指南
- [[docs/agent/testing]] — 测试策略
- [[dev/modules]] — Prism 全局模块结构
- [[recognition]] — Recognition 模块
- [[pipeline]] — Pipeline 模块
- [[channel]] — Channel 模块
