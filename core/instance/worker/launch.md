---
layer: core
source: I:/code/Prism/include/prism/instance/worker/launch.hpp
title: launch — 会话启动与连接分发
module: instance
tags: [instance, worker, launch, session-create, coroutine]
updated: 2026-05-27
---

# launch — 会话启动与连接分发

会话启动模块。当 acceptor 接收新连接后，通过本模块完成 socket 预配置、会话创建和认证设置。分发函数支持跨线程将 socket 投递到目标 worker 的事件循环。

## 核心函数

| 函数 | 说明 |
|------|------|
| `migrate_executor(sock, target_ioc)` | 迁移 socket 的 executor 到目标 io_context |
| `prime(socket, buffer_size)` | 预配置 TCP 参数（TCP_NODELAY + 缓冲区），忽略错误 |
| `start(params)` | 创建传输层 → 构建会话 → 启动协程 |
| `dispatch(params)` | `post` 到目标 io_context → `prime` → `start` |

### launch_params

| 字段 | 类型 | 说明 |
|------|------|------|
| `server` | `server_context&` | 全局配置 |
| `worker` | `worker_context&` | 线程局部上下文 |
| `metrics` | `worker_load&` | 统计状态 |
| `socket` | `tcp::socket` | 已连接的 socket |

## 设计决策

### 为什么 migrate_executor 要释放原生句柄再重新绑定？

`tcp::socket` 被构造时绑定了特定的 `io_context`。`std::move` 只转移所有权，不改变 executor。后续所有异步操作仍通过原 `io_context` 调度。释放 native handle 再用目标 `io_context` 重新 `assign`，不关闭 TCP 连接，仅改变 Asio 的调度路径。

**后果**: assign 失败时需要 `closesocket(native_handle)` 防止 fd 泄漏。

### 为什么 prime() 忽略所有错误？

`setsockopt` 在某些平台（如 Windows）可能不支持部分选项。即使设置失败，连接仍然可用，只是性能略有下降。代理服务的可用性优先级高于微优化。

**后果**: 不检查 `set_option` 的 `error_code`。

### 为什么 dispatch 用 post 而非 strand？

每个 worker 只有一个线程运行 `io_context`，不存在多线程竞争。`post` 将任务投递到目标 io_context 的队列，由其所在线程执行。strand 用于多线程共享同一 io_context 的场景，此处不需要。

**后果**: dispatch 是线程安全的（`post` 本身线程安全），但 start 及后续操作假设在单线程中执行。

## 约束

### dispatch 中 session_open/session_close 必须配对

**类型**: 状态前置

**规则**: `start()` 成功后调用 `metrics.session_open()`，异常时调用 `metrics.session_close()` 补偿。如果 `session_open` 未调用就调用了 `session_close`，计数器会变为负数。

**违反后果**: 活跃会话计数器错误，balancer 负载感知失真。

**源码依据**: `launch.hpp:91` dispatch 注释

### migrate_executor 失败时的 fd 泄漏风险

**类型**: 资源管理

**规则**: `migrate_executor` 释放 native handle 后如果 `assign` 失败，必须 `closesocket`。否则 fd 泄漏。

**违反后果**: 文件描述符逐渐耗尽，最终无法接受新连接。

## 处理流程

```
listener accept → balancer.select → worker.dispatch_socket
                                              │
                                              ▼
                                     dispatch(params)
                                              │
                                     post 到 worker io_context
                                              │
                                              ▼
                                     handoff_pop → prime → start
                                              │
                                              ▼
                                     make_reliable(socket)
                                              │
                                              ▼
                                     make_session(params)
                                              │
                                     ┌────────┴────────┐
                                     │ set_account_directory
                                     │ set_on_closed(callback)
                                     │ session->start()
                                     └────────┬────────┘
                                              │
                                     session_open() 统计
```

## 引用关系

### 依赖

- [[core/context/context|context]]：server/worker 上下文
- [[core/transport/reliable|reliable]]：`make_reliable` 创建传输层
- [[core/stats/runtime|stats::runtime]]：`worker_load` 统计

### 被引用

- [[core/instance/worker/worker|worker]]：`dispatch_socket()` 调用
- [[core/instance/front/listener|listener]]：accept 后通过 balancer 分发
