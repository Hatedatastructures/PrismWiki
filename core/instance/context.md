---
title: "context.hpp — Agent 上下文类型定义（Core 层）"
layer: core
source: "I:/code/Prism/include/prism/instance/context.hpp"
created: 2026-05-17
tags: [agent, context, server_context, worker_context, session_context, PMR]
---

# context.hpp — Agent 上下文类型定义

> 源码: `include/prism/instance/context.hpp`
> 模块: [[overview|Agent]]

## 文件结构

| 类型 | 文件路径 |
|------|----------|
| 头文件 | `include/prism/instance/context.hpp` |
| 源文件 | header-only |

## 模块职责

声明 Agent 运行时上下文结构，贯穿整个请求处理生命周期：

- **server_context**: 服务器全局共享资源
- **worker_context**: 工作线程局部资源
- **session_context**: 会话请求局部资源

为各层组件提供配置、资源和状态访问入口。

## 依赖关系

```
context.hpp
    │
    ├── prism/agent/config.hpp        → agent::config
    ├── prism/config.hpp              → psm::config (顶层配置)
    ├── prism/resolve/router.hpp      → resolve::router
    ├── prism/agent/account/entry.hpp → account::entry, account::lease
    ├── prism/memory/pool.hpp         → memory::resource_pointer, memory::frame_arena
    ├── prism/channel/transport/transmission.hpp → shared_transmission
    ├── prism/outbound/proxy.hpp      → outbound::proxy
    │
    └── boost/asio.hpp                → net::io_context
    └── boost/asio/ssl.hpp            → ssl::context
```

### 调用者

| 调用者 | 用途 |
|--------|------|
| [[worker/worker|worker.hpp]] | 持有 server_context + worker_context |
| [[worker/launch|launch.hpp]] | 构造 session_params 传递上下文 |
| [[session/session|session.hpp]] | 持有 session_context |
| [[dispatch/table|dispatch]] | 从 session_context 获取资源 |

### 被调用者

上下文结构被以下模块使用：

- `psm::config` — server_context.cfg 持有
- `resolve::router` — worker_context.router 引用
- `account::directory` — server_context.account_store 持有
- `transport` — session_context.inbound/outbound 持有

---

## 结构体详解

### server_context

**源码位置**: `context.hpp:48-71`

```cpp
struct server_context
{
    std::atomic<std::shared_ptr<const psm::config>> cfg;    // 配置对象（可原子交换）
    std::shared_ptr<ssl::context> ssl_ctx;             // SSL 上下文
    std::shared_ptr<account::directory> account_store; // 账户注册表

    [[nodiscard]] auto config() const -> const psm::config &
    {
        return *cfg.load();
    }

    void swap_config(std::shared_ptr<const psm::config> new_cfg)
    {
        cfg.store(std::move(new_cfg));
    }
};
```

#### 职责

聚合服务器级别的共享资源，在服务器启动时创建，被所有工作线程共享。

#### 成员详解

| 类型 | 成员 | 说明 |
|------|------|------|
| `std::atomic<std::shared_ptr<const psm::config>>` | `cfg` | 配置对象，原子交换支持热加载 |
| `std::shared_ptr<ssl::context>` | `ssl_ctx` | SSL 上下文，TLS 连接复用 |
| `std::shared_ptr<account::directory>` | `account_store` | 账户注册表，认证管理 |

#### 方法详解

##### config() — 无锁读取配置

**源码位置**: `context.hpp:58-61`

```cpp
[[nodiscard]] auto config() const -> const psm::config &
{
    return *cfg.load();
}
```

逐行解释：

```cpp
[[nodiscard]] auto config() const -> const psm::config &
```
- `[[nodiscard]]`: 警告调用者不要忽略返回值
- `auto ... -> const psm::config &`: 封返回类型，返回配置常量引用
- `const`: 方法不修改成员状态

```cpp
{
    return *cfg.load();
}
```
- `cfg.load()`: 原子读取 shared_ptr（无锁，仅原子读取）
- `*`: 解引用，获取 psm::config 对象
- 返回引用，避免拷贝开销

##### swap_config() — 原子交换配置

**源码位置**: `context.hpp:67-70`

```cpp
void swap_config(std::shared_ptr<const psm::config> new_cfg)
{
    cfg.store(std::move(new_cfg));
}
```

逐行解释：

```cpp
void swap_config(std::shared_ptr<const psm::config> new_cfg)
```
- 参数类型 `shared_ptr<const psm::config>`: 共享所有权配置指针
- `const`: 配置内容不可修改，确保运行时一致性

```cpp
{
    cfg.store(std::move(new_cfg));
}
```
- `std::move(new_cfg)`: 转移所有权，避免拷贝
- `cfg.store(...)`: 原子存储，支持热加载时无锁切换

#### 调用链

```
worker::worker(cfg, account_store)  // worker.cpp:11-45
    │
    ├── 构造 server_context
    │       server_ctx_{
    │           std::atomic<std::shared_ptr<const psm::config>>{
    │               std::make_shared<const psm::config>(cfg)
    │           },
    │           ssl_ctx_,
    │           std::move(account_store)
    │       }
    │
    ▼
server_context 持有:
    ├── cfg.load() → psm::config
    ├── ssl_ctx → ssl::context (TLS 服务端)
    └── account_store → account::directory (认证注册表)
```

---

### worker_context

**源码位置**: `context.hpp:84-90`

```cpp
struct worker_context
{
    net::io_context &io_context;          // I/O 上下文引用
    resolve::router &router;              // 路由器引用
    memory::resource_pointer memory_pool; // 内存池资源指针
    outbound::proxy *outbound{nullptr};   // 出站代理指针（由 worker 拥有）
};
```

#### 职责

封装单个工作线程的独立资源，实现线程间资源隔离和避免锁竞争。

#### 成员详解

| 类型 | 成员 | 说明 |
|------|------|------|
| `net::io_context &` | `io_context` | 事件循环引用，生命周期由 worker 管理 |
| `resolve::router &` | `router` | DNS 路由器引用，建立上游连接 |
| `memory::resource_pointer` | `memory_pool` | PMR 内存池指针，热路径分配 |
| `outbound::proxy *` | `outbound` | 出站代理指针（非拥有，由 worker 拥有） |

#### 设计要点

- io_context 单线程运行 (`ioc_(1)`)，避免锁竞争
- router 在 worker 线程内独享，无跨线程访问
- memory_pool 使用 `thread_local_pool`，线程局部
- outbound 由 worker 拥有，worker_context 仅持有裸指针

#### 调用链

```
worker::worker() 构造
    │
    ├── ioc_(1)                    // 单线程 io_context
    ├── pool_(ioc_, memory::system::thread_local_pool(), cfg.pool)
    ├── router_(pool_, ioc_, cfg.dns, memory::system::thread_local_pool())
    │
    ├── 构造 worker_context
    │       worker_ctx_{
    │           ioc_,                                      // io_context 引用
    │           router_,                                   // router 引用
    │           memory::system::thread_local_pool(),       // PMR 池
    │           outbound_direct_.get()                     // 出站代理指针
    │       }
    │
    ▼
worker_context 在会话中使用:
    │
    ├── session::start() → ctx_.worker.io_context.post()
    ├── pipeline::dial() → ctx_.worker.router.resolve()
    ├── connect::allocate() → ctx_.worker.memory_pool
```

---

### session_context

**源码位置**: `context.hpp:105-132`

```cpp
struct session_context
{
    session_context(const session_context &) = delete;
    session_context &operator=(const session_context &) = delete;
    session_context(session_context &&) = default;
    session_context &operator=(session_context &&) = delete;

    session_context(const std::uint64_t sid, const server_context &srv, worker_context &w,
                    memory::frame_arena &arena, account::directory *dir, std::function<bool(std::string_view)> verifier,
                    const std::uint32_t buf_size, shared_transmission in)
        : session_id(sid), server(srv), worker(w), frame_arena(arena),
          credential_verifier(std::move(verifier)), account_directory_ptr(dir),
          buffer_size(buf_size), inbound(std::move(in)) {}

    std::uint64_t session_id{0};                               // 会话唯一标识符
    const server_context &server;                              // 服务器上下文常量引用
    worker_context &worker;                                    // 工作线程上下文引用
    memory::frame_arena &frame_arena;                          // 帧内存池引用
    std::function<bool(std::string_view)> credential_verifier; // 凭据验证函数
    account::directory *account_directory_ptr{nullptr};        // 账户注册表指针
    std::uint32_t buffer_size;                                 // 数据传输缓冲区大小（字节）
    shared_transmission inbound;                               // 入站传输对象
    shared_transmission outbound;                              // 出站传输对象
    outbound::proxy *outbound_proxy{nullptr};                  // 出站代理指针（由 worker 设置）
    account::lease account_lease;                              // 账户连接租约
    std::function<void()> active_stream_cancel;                // 活跃流取消回调
    std::function<void()> active_stream_close;                 // 活跃流关闭回调
};
```

#### 职责

聚合单个连接会话所需的所有资源和状态，是请求处理流程的核心数据结构。

#### 成员详解

| 类型 | 成员 | 说明 |
|------|------|------|
| `std::uint64_t` | `session_id` | 会话唯一标识符，全局递增 |
| `const server_context &` | `server` | 服务器上下文常量引用 |
| `worker_context &` | `worker` | 工作线程上下文引用 |
| `memory::frame_arena &` | `frame_arena` | 帧内存池引用，会话结束回收 |
| `std::function<bool(std::string_view)>` | `credential_verifier` | 凭据验证回调（SOCKS5/HTTP 认证） |
| `account::directory *` | `account_directory_ptr` | 账户注册表指针（连接配额） |
| `std::uint32_t` | `buffer_size` | 数据传输缓冲区大小 |
| `shared_transmission` | `inbound` | 入站传输对象（客户端连接） |
| `shared_transmission` | `outbound` | 出站传输对象（上游连接） |
| `outbound::proxy *` | `outbound_proxy` | 出站代理指针（由 worker 设置） |
| `account::lease` | `account_lease` | 账户连接租约（配额管理） |
| `std::function<void()>` | `active_stream_cancel` | 多路复用流取消回调 |
| `std::function<void()>` | `active_stream_close` | 多路复用流关闭回调 |

#### 构造函数详解

**源码位置**: `context.hpp:112-117`

```cpp
session_context(const std::uint64_t sid, const server_context &srv, worker_context &w,
                memory::frame_arena &arena, account::directory *dir, std::function<bool(std::string_view)> verifier,
                const std::uint32_t buf_size, shared_transmission in)
    : session_id(sid), server(srv), worker(w), frame_arena(arena),
      credential_verifier(std::move(verifier)), account_directory_ptr(dir),
      buffer_size(buf_size), inbound(std::move(in)) {}
```

逐行解释：

```cpp
session_context(const std::uint64_t sid, const server_context &srv, worker_context &w,
                memory::frame_arena &arena, account::directory *dir,
                std::function<bool(std::string_view)> verifier,
                const std::uint32_t buf_size, shared_transmission in)
```
- `sid`: 会话 ID，由 `detail::generate_session_id()` 生成
- `srv`: server_context 常量引用，生命周期由 worker 管理
- `w`: worker_context 引用，同一线程内
- `arena`: 帧内存池引用，由 session 持有的 `frame_arena_`
- `dir`: 账户注册表指针，用于连接配额检查
- `verifier`: 凭据验证回调，SOCKS5/HTTP 认证时使用
- `buf_size`: 缓冲区大小，来自 `cfg.buffer.size`
- `in`: 入站传输对象，所有权转移

```cpp
: session_id(sid), server(srv), worker(w), frame_arena(arena),
  credential_verifier(std::move(verifier)), account_directory_ptr(dir),
  buffer_size(buf_size), inbound(std::move(in)) {}
```
- 初始化列表成员初始化
- `std::move(verifier)`: 转移验证回调所有权
- `std::move(in)`: 转移入站传输所有权

#### 禁止拷贝设计

```cpp
session_context(const session_context &) = delete;
session_context &operator=(const session_context &) = delete;
session_context(session_context &&) = default;
session_context &operator=(session_context &&) = delete;
```

- 禁止拷贝：引用成员不可拷贝
- 允许移动：支持从 session_params 转移
- 禁止移动赋值：避免已初始化对象被覆盖

#### 调用链详解

```
launch::dispatch(ioc, server_ctx, worker_ctx, metrics, socket)  // launch.cpp
    │
    ├── 创建传输层
    │       auto transmission = transport::make_reliable(socket, ...);
    │
    ├── 构造 session_params
    │       session_params params{
    │           server_ctx,    // server_context 引用
    │           worker_ctx,    // worker_context 引用
    │           std::move(transmission)  // 入站传输所有权
    │       };
    │
    ├── 创建会话
    │       auto sess = make_session(std::move(params));
    │           │
    │           ▼
    │       session::session(params)
    │           │
    │           ├── generate_session_id() → id_
    │           ├── 构造 frame_arena_
    │           │
    │           ├── 构造 session_context
    │           │       ctx_{
    │           │           id_,                    // session_id
    │           │           params.server,          // server 引用
    │           │           params.worker,          // worker 引用
    │           │           frame_arena_,           // 帧内存池
    │           │           nullptr,                // account_directory_ptr（后续设置）
    │           │           nullptr,                // credential_verifier（后续设置）
    │           │           cfg.buffer.size,        // buffer_size
    │           │           std::move(params.inbound) // inbound
    │           │       }
    │
    ├── 设置验证器和账户目录
    │       sess->set_credential_verifier(verifier);
    │       sess->set_account_directory(account_store);
    │
    ├── 启动会话
    │       sess->start();
    │           │
    │           ▼
    │       session::diversion() 协程
    │           │
    │           ├── ctx_.server.config() → psm::config
    │           ├── ctx_.worker.router → resolve::router
    │           ├── ctx_.frame_arena → PMR 临时分配
    │           ├── ctx_.inbound → 协议检测
    │           ├── ctx_.credential_verifier → 认证验证
```

---

## 上下文层次关系

```
server_context (全局共享，启动时创建)
    │
    ├── cfg → psm::config (原子指针，支持热加载)
    ├── ssl_ctx → ssl::context (TLS 服务端配置)
    └── account_store → account::directory (认证注册表)
    │
    ▼ ──── 由 worker 持有
worker_context (线程局部，每个 worker 独享)
    │
    ├── io_context → 事件循环 (单线程)
    ├── router → DNS 路由器 (线程独享)
    ├── memory_pool → PMR 线程池
    └── outbound → 出站代理指针
    │
    ▼ ──── 由 session 持有
session_context (会话局部，每个连接独立)
    │
    ├── session_id → 全局唯一标识
    ├── server → server_context 引用
    ├── worker → worker_context 引用
    ├── frame_arena → 帧内存池 (会话结束回收)
    ├── inbound → 入站传输 (客户端)
    ├── outbound → 出站传输 (上游)
    ├── credential_verifier → 认证回调
    ├── account_lease → 连接租约
    │
    ▼ ──── 会话结束后销毁
```

---

## 设计要点

### 原子配置交换

server_context 使用 `std::atomic<std::shared_ptr>` 持有配置：

- 无锁读取 (`cfg.load()`)
- 原子交换 (`cfg.store()`)
- 支持配置热加载（未来功能）

### 线程隔离

worker_context 成员全为引用/指针：

- io_context 单线程运行，无锁竞争
- router 线程独享，DNS 缓存线程局部
- memory_pool 线程局部，避免池竞争

### 会话生命周期

session_context 禁止拷贝：

- 引用成员不可拷贝
- 移动构造支持资源转移
- 会话结束自动回收 frame_arena

---

## 知识域

- [[config|配置类型定义]] — server_context.cfg 持有的类型
- [[worker/worker|Worker]] — 持有 server_context + worker_context
- [[session/session|会话]] — 持有 session_context
- [[account/directory|账户目录]] — server_context.account_store 类型
- [[core/resolve/router|DNS 路由器]] — worker_context.router 类型
- [[core/transport/transmission|传输层]] — session_context.inbound/outbound 类型
- [[core/memory/pool|PMR 内存池]] — frame_arena 类型
- [[outbound/proxy|出站代理]] — outbound_proxy 类型