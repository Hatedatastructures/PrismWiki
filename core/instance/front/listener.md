---
title: listener — 前端监听器
layer: core
source: include/prism/instance/front/listener.hpp
created: 2026-05-17
tags: [agent, front, listener, accept, affinity]
---

# listener — 前端监听器

> 源码: `include/prism/instance/front/listener.hpp`

## 概述

`listener` 是 Prism 代理服务器的入站阶段入口组件，位于 **Front Layer** 的最顶层。负责监听指定端口、接受入站 TCP 连接，并将连接分发给负载均衡器进行后续处理。

### 核心职责

| 职责 | 说明 |
|------|------|
| TCP 监听 | 绑定地址端口，持续异步接受连接 |
| 亲和性计算 | 根据客户端 IP 计算一致性哈希值 |
| 连接分发 | 调用 balancer 的 `select()` 和 `dispatch()` |
| 反压处理 | 系统过载时延迟接受新连接 |

### 调用关系

```
[main.cpp] ──构造──► [[listener]] ──listen()──► accept_loop协程
                                  │
                                  ├─► make_affinity()
                                  │
                                  └─► [[core/instance/front/balancer|balancer]].select()
                                          │
                                          └─► [[core/instance/front/balancer|balancer]].dispatch()
                                                  │
                                                  └─► [[core/instance/worker/worker|worker]].dispatch_socket()
```

---

## 类定义

```cpp
namespace psm::instance::front
{
    class listener
    {
    public:
        explicit listener(const psm::config &cfg, balancer &dispatcher);
        void listen();

    private:
        [[nodiscard]] static auto make_affinity(const tcp::endpoint &endpoint) noexcept -> std::uint64_t;
        auto accept_loop() -> net::awaitable<void>;

        net::io_context ioc_;                          // 独立的 IO 上下文
        tcp::acceptor acceptor_;                       // TCP 接受器
        balancer &dispatcher_;                         // 负载均衡器引用
        std::uint32_t buffer_size_;                    // 套接字缓冲区大小
        std::chrono::milliseconds backpressure_delay_; // 反压延迟时间
    };
}
```

### 成员变量详解

| 成员 | 类型 | 说明 |
|------|------|------|
| `ioc_` | `net::io_context` | 独立事件循环，与 worker 隔离 |
| `acceptor_` | `tcp::acceptor` | TCP 接受器，绑定监听端口 |
| `dispatcher_` | `balancer &` | 负载均衡器引用 |
| `buffer_size_` | `uint32_t` | Socket 收发缓冲区大小（默认 256KB） |
| `backpressure_delay_` | `milliseconds` | 反压延迟时间（固定 2ms） |

---

## 函数详解

### listener() 构造函数

```cpp
explicit listener(const psm::config &cfg, balancer &dispatcher);
```

**参数**：
- `cfg`: 全局配置，包含监听地址、端口、缓冲区大小
- `dispatcher`: 负载均衡器引用

**执行流程**（逐行解释）：

```cpp
// 1. 初始化 io_context（单线程）
ioc_(1)

// 2. 初始化 acceptor 绑定到 io_context
acceptor_(ioc_)

// 3. 存储配置参数
buffer_size_ = cfg.buffer.size          // 缓冲区大小
dispatcher_ = dispatcher                 // 负载均衡器引用

// 4. 解析监听地址
net::ip::address addr = net::ip::make_address(cfg.agent.addressable.host, ec)
if (ec):
    if host == "localhost": addr = 127.0.0.1       // IPv4 回环
    elif host == "0.0.0.0" or empty: addr = 0.0.0.0 // 所有接口
    else: throw std::system_error                   // 无效地址

// 5. 构建端点
endpoint = {addr, cfg.agent.addressable.port}

// 6. 配置 acceptor
acceptor_.open(endpoint.protocol())               // 打开 IPv4 socket
acceptor_.set_option(reuse_address(true))         // 允许端口复用
acceptor_.set_option(receive_buffer_size(buffer_size_))
acceptor_.set_option(send_buffer_size(buffer_size_))
acceptor_.bind(endpoint)                          // 绑定地址端口
acceptor_.listen()                                // 开始监听（backlog 由系统决定）

// 7. 初始化反压延迟
backpressure_delay_ = 2ms                         // 固定 2 毫秒
```

**设计要点**：
- 使用独立 `io_context` 避免监听线程被 worker 负载阻塞
- `reuse_address(true)` 允许 TIME_WAIT 状态端口复用
- 仅支持 IPv4 地址，IPv6 计划后续支持

---

### listen() 函数

```cpp
void listen();
```

**职责**: 启动监听，派生 accept 循环协程，阻塞运行 io_context。

**逐行解释**：

```cpp
// 1. 派生 accept_loop 协程
net::co_spawn(ioc_, accept_loop(), net::detached)
// 使用 detached 表示协程结果不需要等待
// accept_loop 在 ioc_ 中运行

// 2. 阻塞运行事件循环
ioc_.run()
// 阻塞调用线程，通常在专用监听线程中调用
// accept_loop 协程无限循环，直到 io_context 停止
```

**调用者**: `main.cpp` 在独立线程调用：

```cpp
auto listen_thread = [&service_listener]() {
    service_listener.listen();  // 阻塞运行
};
threads.emplace_back(listen_thread);
```

---

### make_affinity() 函数（私有静态）

```cpp
[[nodiscard]] static auto make_affinity(const tcp::endpoint &endpoint) noexcept -> std::uint64_t;
```

**职责**: 根据客户端端点计算亲和性哈希值，用于一致性选择。

**逐行解释**：

```cpp
// 获取客户端 IP 地址
const auto &address = endpoint.address()

// 判断地址类型
if (address.is_v4()):
    // IPv4: 直接返回 32 位地址值
    return address.to_v4().to_uint()
    // 例: 192.168.1.1 → 0xC0A80101 (3232235777)

else if (address.is_v6()):
    // IPv6: 高低位 XOR 折叠为 64 位
    const auto bytes = address.to_v6().to_bytes()    // 16 字节数组
    const uint64_t high = bytes[0..7]                 // 高 8 字节
    const uint64_t low = bytes[8..15]                 // 低 8 字节
    return high ^ low                                 // 折叠为 64 位
```

**设计意图**：
- 相同客户端 IP 总是产生相同的亲和性值
- balancer 基于此值进行一致性哈希选择
- 同一客户端连接倾向于分发到同一 worker
- 端口号未参与计算，仅使用 IP 地址

---

### accept_loop() 协程（私有）

```cpp
auto accept_loop() -> net::awaitable<void>;
```

**职责**: 持续接受入站连接，计算亲和性，分发到 balancer。

**逐行解释**：

```cpp
// ===== 初始化阶段 =====

// 1. 获取协程执行器
executor = co_await net::this_coro::executor

// 2. 创建定时器（用于延迟）
timer = steady_timer{executor}

// 3. 初始化指数退避参数（静态局部变量）
static constexpr min_delay = 10ms      // 最小延迟
static constexpr max_delay = 5120ms    // 最大延迟（约 5 秒）
static thread_local delay = min_delay  // 当前延迟（线程局部）

// ===== 接受循环 =====

while (true):  // 无限循环
    
    // 4. 异步接受连接
    [socket, ec] = co_await acceptor_.async_accept(net::redirect_error)
    
    // 5. 错误处理
    if (ec):
        if (致命错误: too_many_files_open, not_enough_memory):
            // 指数退避
            timer.expires_after(delay)
            co_await timer.async_wait()
            delay = min(delay * 2, max_delay)  // 延迟翻倍，上限 5120ms
            continue                            // 重试
        
        else (普通错误):
            timer.expires_after(10ms)           // 固定延迟
            co_await timer.async_wait()
            continue                            // 重试
    
    // ===== 成功接受 =====
    
    // 6. 重置退避延迟（成功后恢复）
    delay = min_delay
    
    // 7. 获取客户端端点
    remote_endpoint = socket.remote_endpoint()
    
    // 8. 计算亲和性值
    affinity = make_affinity(remote_endpoint)
    
    // 9. 选择目标 worker
    decision = dispatcher_.select(affinity)
    // 返回 {worker_index, overflowed, backpressure}
    
    // 10. 处理反压
    if (decision.backpressure):
        timer.expires_after(backpressure_delay_)  // 2ms
        co_await timer.async_wait()
        // 延迟后继续分发（不拒绝连接）
    
    // 11. 分发连接
    dispatcher_.dispatch(decision.worker_index, std::move(socket))
    // socket 所有权转移给 worker
    
    // 12. 继下一轮循环
```

**关键设计点**：

| 设计点 | 说明 |
|--------|------|
| 指数退避 | 致命错误时延迟从 10ms 翻倍至 5120ms，避免 CPU 空转 |
| 反压机制 | 系统过载时延迟 2ms 后继续，平滑处理而非拒绝 |
| socket 转移 | 使用 `std::move()` 将所有权移交 worker |
| 协程纯度 | 禁止阻塞操作，使用 `steady_timer` 实现延迟 |

---

## 调用链

### 上游调用者

| 调用者 | 调用时机 |
|--------|----------|
| `main.cpp` | 启动时构造实例 |
| `main.cpp` | 监听线程中调用 `listen()` |

### 下游依赖

| 依赖模块 | 调用方式 |
|----------|----------|
| [[core/instance/front/balancer|balancer]] | `dispatcher_.select(affinity)` |
| [[core/instance/front/balancer|balancer]] | `dispatcher_.dispatch(index, socket)` |

### 完整调用链

```
main.cpp:97  listener service_listener(config_ref, dispatcher)
    │
    ▼
main.cpp:128  service_listener.listen()
    │
    ▼
listener.cpp:51  net::co_spawn(ioc_, accept_loop(), net::detached)
    │
    ▼
listener.cpp:125  dispatcher_.select(affinity)
    │               → balancer::select()
    │               → balancer::score()
    │               → balancer::refresh_state()
    │
    ▼
listener.cpp:135  dispatcher_.dispatch(worker_index, socket)
    │               → balancer::dispatch()
    │               → worker_binding.dispatch(socket)
    │               → [[core/instance/worker/worker|worker]].dispatch_socket()
```

---

## 设计决策

### 独立 io_context

**选择**: listener 使用独立 `io_context(1)`，而非共享 worker 的。

**理由**:
- 监听线程不受 worker 处理负载阻塞
- worker 过载不会影响 accept 能力
- socket 跨线程转移增加开销，但保证稳定性

### 协程模式

**选择**: 协程异步 accept，而非传统回调。

**理由**:
- 代码流程清晰，易于理解
- 错误处理统一，避免回调地狱
- 与 Prism 整体架构一致（纯协程）

### 反压延迟

**选择**: 固定 2ms 延迟，而非指数退避或拒绝。

**理由**:
- 2ms 足够短，不影响正常流量
- 足够长，让过载 worker 有恢复机会
- 避免"拒绝-重试"导致的客户端重连风暴

---

## 错误处理

### accept 错误类型

| 错误类型 | 处理策略 | 延迟 |
|----------|----------|------|
| `too_many_files_open` | 指数退避 | 10ms → 5120ms |
| `not_enough_memory` | 指数退避 | 10ms → 5120ms |
| 其他错误 | 固定延迟 | 10ms |

### 异常抛出

| 场景 | 异常类型 |
|------|----------|
| 地址解析失败 | `std::system_error` |
| 端口被占用 | `std::system_error` |
| 权限不足 | `std::system_error` |

---

## 相关文档

- [[core/instance/front/balancer|balancer]] — 负载均衡器
- [[core/instance/worker/worker|worker]] — 工作线程
- [[core/instance/config|config]] — 配置详解
- [[startup]] — 启动流程