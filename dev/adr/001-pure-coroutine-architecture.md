---
title: "ADR 001: 选择纯协程架构"
created: 2026-05-23
updated: 2026-05-23
layer: dev
tags: [dev, adr, architecture, coroutine]
status: accepted
---

# ADR 001: 选择纯协程架构

## 状态

已接受 (Accepted)

## 背景

Prism 作为高性能代理服务器，需要同时处理大量并发网络连接。在架构选型阶段，存在三种主要的异步编程模型：

### 方案 A: 线程池 + 阻塞 I/O

每个连接分配一个线程（或从线程池获取），使用阻塞式 socket 操作。

- **优势**: 编程模型直观，代码可读性好，调试方便
- **劣势**: 线程是操作系统资源，每个线程占用 MB 级栈空间；线程切换成本高（上下文切换、缓存失效）；C10K 问题严重；锁竞争导致性能下降

### 方案 B: 回调 (Callback) + 事件循环

基于 Boost.Asio 的传统异步模型，通过回调函数处理 I/O 完成。

- **优势**: 单线程可处理大量连接，内存占用低
- **劣势**: 回调地狱（Callback Hell），控制流分散在多个回调中，代码难以理解和维护；错误处理复杂；资源管理容易出错

### 方案 C: C++20/23 协程 + Boost.Asio

使用 C++ 协程（`co_await`/`co_return`）配合 Boost.Asio 的异步操作，以同步的代码风格实现异步 I/O。

- **优势**: 代码可读性接近同步模型，同时具备异步的高性能；零拷贝传递；编译器优化友好
- **劣势**: C++ 协程生态仍在发展中，调试工具不成熟；编译时间增加；协程安全性需要严格遵守规范

## 决策

**选择方案 C: C++23 协程 + Boost.Asio**

具体技术选型：

- **协程框架**: `net::awaitable<T>`（`namespace net = boost::asio`）
- **异步操作**: 所有 I/O 操作返回 `net::awaitable<T>`，使用 `co_await` 挂起/恢复
- **协程启动**: 使用 `net::co_spawn` 启动独立协程
- **线程模型**: 每个 worker 线程运行独立的 `io_context`，线程间无共享状态
- **同步原语**: 禁止 `std::mutex`/`std::lock_guard`，使用 `std::atomic`、`strand`、`concurrent_channel`

### 纯度要求

Prism 采用**纯协程架构**，在协程中禁止任何阻塞操作：

| 禁止 | 替代方案 |
|------|----------|
| `std::mutex` / `std::lock_guard` | `std::atomic`、`strand`、`concurrent_channel` |
| `std::this_thread::sleep_for()` | `net::steady_timer::async_wait()` |
| 阻塞 socket read/write | `async_read_some` / `async_write_some` |
| `::getaddrinfo()` 同步 DNS | `resolver.async_resolve()` |
| `std::future::get()` / `wait()` | `co_await` 异步结果 |
| `while (!flag) {}` 忙等待 | `co_await` 异步等待 + 通知机制 |

### 架构层次

```
Front Layer (listener/balancer)
    |
    v
Worker Layer (每线程 io_context)
    |
    v
Session Layer (session 生命周期)
    |
    v
Recognition Layer (probe -> identify -> execute)
    |
    v
Pipeline Layer (协议处理)
    |
    v
Channel Layer (连接池/传输)
```

每一层都是纯协程，从上到下贯穿 `co_await` 调用链。

> 参考: [[dev/coding/coroutine|协程约定]]、[[dev/modules|模块架构]]

### 具体实现细节

#### 协程在 Prism 中的使用模式

**模式 1: 顺序异步操作**

最常见的模式，使用 `co_await` 按顺序执行多个异步操作：

```cpp
auto session::run() -> net::awaitable<void> {
    auto [ec, result] = co_await recognition::recognize({...});
    if (ec != fault::code::success) co_return;

    auto handler = dispatch::registry::global().create(result.detected);
    co_await handler->handle(result.transport, result.preread);
}
```

**模式 2: 独立协程启动**

使用 `net::co_spawn` 启动独立的并行协程，常见于隧道双向转发：

```cpp
auto tunnel(net::awaitable<void> copy_a_to_b,
            net::awaitable<void> copy_b_to_a) -> net::awaitable<void> {
    // 两个方向同时启动
    auto ex = co_await net::this_coro::executor;
    net::co_spawn(ex, std::move(copy_a_to_b), net::detached);
    co_await copy_b_to_a;
}
```

**模式 3: 超时控制**

使用 `net::steady_timer` 实现异步超时，替代 `std::this_thread::sleep_for`：

```cpp
auto with_timeout(net::awaitable<T> op, uint32_t ms) -> net::awaitable<T> {
    auto ex = co_await net::this_coro::executor;
    net::steady_timer timer(ex, std::chrono::milliseconds(ms));
    // 使用 async_compose 或 cancellation_slot 实现超时取消
    co_return co_await std::move(op);
}
```

#### Worker 线程模型

每个 worker 线程运行独立的 `io_context`，协程天然绑定到创建它的 `io_context`：

```
Thread 0 (Listener):
  io_context_0
    |- listener::accept()   // 接受新连接
    |- balancer::dispatch() // 分发到 worker

Thread 1 (Worker):
  io_context_1
    |- session_1::run()     // 协程，处理连接 1
    |- session_2::run()     // 协程，处理连接 2
    |- session_N::run()     // 协程，处理连接 N
    |- connection_pool_1    // 线程局部连接池

Thread K (Worker):
  io_context_K
    |- session_1::run()
    |- ...
```

线程间通过 `post()` 跨 `io_context` 投递任务，不使用共享队列或互斥锁。

#### 与 Go goroutine 的对比

虽然 Prism 的协程架构灵感部分来自 Go 的 goroutine 模型，但存在关键差异：

| 维度 | Go goroutine | Prism 协程 |
|------|-------------|-----------|
| 调度器 | 运行时 M:N 调度 | 1:1 绑定 io_context |
| 栈管理 | 动态伸缩（2KB 起） | 编译器固定大小帧 |
| 抢占 | 异步抢占（since 1.14） | 协作式（co_await 点） |
| 通道 | 内置 chan | concurrent_channel |
| 锁 | 允许 sync.Mutex | 禁止 std::mutex |
| GC | 自动 | 手动 PMR |

Prism 选择协作式协程而非 Go 的抢占式 goroutine，是因为代理服务器的 I/O 边界天然就是 `co_await` 点，不需要运行时调度器的复杂度。

## 后果

### 优势

1. **可读性**: 协程代码以线性风格编写，控制流清晰可见。对比回调模型，`co_await` 让异步代码的阅读体验接近同步代码。例如隧道双向转发的实现就是简洁的 `co_await` 循环。

2. **性能**:
   - 协程挂起/恢复成本远低于线程切换（纳秒级 vs 微秒级）
   - 协程帧大小由编译器自动计算，通常只需数百字节
   - 单个 `io_context` 线程可处理数万并发连接
   - 配合 PMR 内存策略，热路径实现零堆分配

3. **资源效率**:
   - 无需为每个连接分配线程栈（MB 级）
   - 协程帧按需分配，内存利用率高
   - `io_context` 的 proactor 模式天然支持批量 I/O

4. **错误处理**: 协程支持异常传播，`co_await` 可以自然地传播错误。配合 Prism 的双轨错误处理策略（热路径用 `fault::code`，启动/致命错误用异常），错误处理既高效又安全。

5. **组合性**: 协程天然支持组合，可以通过 `co_await` 串联多个异步操作，也可以通过 `co_spawn` 并行启动多个协程。

6. **无锁设计**: 协程在同一 `io_context` 中顺序执行，天然串行，不需要锁保护共享状态。这消除了锁竞争、死锁等并发编程的经典问题。

7. **确定性调度**: 协作式调度意味着协程只在自己的 `co_await` 点挂起，不会在任意指令处被抢占。这使得对共享状态的推理更加简单。

### 劣势

1. **调试难度**: C++ 协程的调试体验远不如线程模型。协程的调用栈在挂起后不再保留传统栈帧，GDB/LLDB 对协程的支持有限。调试时需要依赖日志和 `spdlog` 的追踪功能。

2. **编译时间**: 协程相关的模板展开增加了编译时间。Boost.Asio 的 `awaitable` 模板实例化在热路径中大量使用，导致编译耗时较长。

3. **协程安全性**: 需要严格遵守纯度要求，任何阻塞调用都会阻塞整个 `io_context` 线程，影响所有在该线程上运行的协程。这需要开发者对协程模型有深入理解。

4. **生命周期复杂性**: `co_await` 挂起后恢复时，之前获取的裸指针、迭代器、引用可能已失效。`co_spawn` 的 lambda 必须按值捕获 `self`（shared_ptr）保持对象存活。`erase()` 后必须使用返回值更新迭代器。

5. **生态成熟度**: C++23 协程相关工具（调试器、分析器、静态分析）仍在发展中。部分第三方库尚未适配协程 API。

6. **阻塞的风险**: 单个协程中的阻塞操作（如文件 I/O、DNS 查询、第三方库的同步 API）会阻塞整个 `io_context` 线程，影响所有其他协程。这种问题难以通过代码审查完全防止。

7. **学习曲线**: C++ 协程的语义（promise_type、co_await 转换规则、协程帧布局）对新手不友好。团队需要投入培训成本。

### 缓解措施

- 编写详尽的日志追踪（`trace` 模块），弥补调试工具不足
- 使用 `[[nodiscard]]` 标注所有有意义的返回值
- 建立严格的代码审查机制，确保协程纯度
- 通过 [[dev/testing/stress|压力测试]] 和 [[dev/testing/benchmark|基准测试]] 验证性能

## 相关页面

- [[dev/coding/coroutine|协程约定]]
- [[dev/coding/lifecycle|生命周期安全]]
- [[dev/modules|模块架构]]
- [[dev/adr/002-pmr-memory-strategy|ADR 002: PMR 内存策略]]
- [[core/overview|Core 层总览]]
