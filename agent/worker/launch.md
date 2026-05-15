---
title: "launch.hpp — 会话启动与连接分发模块"
source: "include/prism/agent/worker/launch.hpp"
module: "agent"
type: api
tags: [agent, worker, launch, 会话启动, socket 迁移]
created: 2026-05-15
updated: 2026-05-15
related:
  - agent/worker/worker
  - agent/session/session
  - agent/worker/stats
  - agent/context
  - channel/transport/reliable
  - agent/account/directory
---

# launch.hpp

> 源码: `include/prism/agent/worker/launch.hpp` + `src/prism/agent/worker/launch.cpp`
> 模块: [[agent|Agent]] / worker / launch

## 概述

提供新连接的预处理和会话启动功能。当 acceptor 接收新连接后，通过本模块完成 socket 预配置、executor 跨线程迁移、会话创建和认证设置等初始化工作。分发函数支持跨线程将 socket 投递到目标 worker 的事件循环中执行，实现负载均衡的连接分发机制。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | server_context / worker_context 定义 |
| 依赖 | [[agent/worker/stats|stats]] | 统计状态（会话计数器、handoff 计数） |
| 依赖 | [[agent/session/session|session]] | 会话创建和启动 |
| 依赖 | [[channel/transport/reliable|reliable]] | 可靠传输层封装 |
| 依赖 | [[agent/account/directory|account]] | 账户目录查询 |
| 依赖 | [[trace|trace]] | 日志输出 |
| 被依赖 | [[agent/worker/worker|worker]] | worker 调用 launch 函数 |

## 命名空间

`psm::agent::worker::launch`

---

## 函数: migrate_executor()

#### 功能说明
将 socket 的 executor 从当前 `io_context` 迁移到目标 `io_context`。socket 被 move 后 executor 不会改变，导致后续异步操作仍在原线程上执行。该函数通过释放原生句柄（`release()`）并重新绑定（`assign()`）到目标 `io_context` 来解决这个问题。

迁移过程：先查询本地端点判断 IPv4/IPv6 协议类型，然后 `release()` 剥离原生句柄使原 socket 变为空壳，最后在目标 `io_context` 上 `assign()` 重建 socket。如果 `assign` 失败，关闭已释放的 native handle 防止文件描述符泄漏。

#### 签名
```cpp
[[nodiscard]] std::optional<tcp::socket> migrate_executor(tcp::socket &sock, net::io_context &target_ioc) noexcept;
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `sock` | `tcp::socket &` | 待迁移的 socket，迁移后变为空壳 |
| `target_ioc` | `net::io_context &` | 目标 io_context，迁移后 socket 的 executor 将绑定到它 |

#### 返回值
| 类型 | 说明 |
|------|------|
| `std::optional<tcp::socket>` | 迁移后的新 socket，失败时返回 `std::nullopt` |

#### 调用（向下）
- `sock.local_endpoint()` — 查询本地端点，判断 IPv4/IPv6
- `sock.release()` — 剥离原生句柄
- `tcp::socket::assign()` — 在目标 io_context 上重新绑定句柄
- `::closesocket()` / `::close()` — assign 失败时关闭 native handle 防止 fd 泄漏

#### 被调用（向上）
- `launch::dispatch()` — 在 worker 线程中迁移 socket

#### 知识域
- [[ref/programming/boost-asio|Boost.Asio socket 迁移]] — `release()` + `assign()` 实现跨 io_context 迁移
- [[ref/network/socket-fd|文件描述符管理]] — 迁移失败时必须关闭已释放的 fd

---

## 函数: prime()

#### 功能说明
预配置 TCP socket 参数，对新接收的 socket 进行性能优化配置。主要设置三项参数：打开 `TCP_NODELAY` 禁用 Nagle 算法以降低小包延迟，设置接收缓冲区大小以匹配应用层吞吐需求，设置发送缓冲区大小以优化数据发送效率。所有操作均忽略错误，因为 socket 配置失败不应阻断连接处理。

#### 签名
```cpp
void prime(tcp::socket &socket, std::uint32_t buffer_size) noexcept;
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `socket` | `tcp::socket &` | 待配置的 TCP socket |
| `buffer_size` | `std::uint32_t` | 接收和发送缓冲区大小（字节），来自 `config.buffer.size` |

#### 返回值
无。

#### 调用（向下）
- `socket.set_option(tcp::no_delay(true))` — 禁用 Nagle 算法
- `socket.set_option(net::socket_base::receive_buffer_size(...))` — 设置接收缓冲区
- `socket.set_option(net::socket_base::send_buffer_size(...))` — 设置发送缓冲区

#### 被调用（向上）
- `launch::dispatch()` — socket 迁移成功后调用

#### 知识域
- [[ref/network/tcp-nagle|TCP_NODELAY]] — 禁用 Nagle 算法减少小包延迟
- [[agent/config|配置体系]] — 缓冲区大小来自 `config.buffer.size`

---

## 函数: start()

#### 功能说明
启动新会话。在 worker 线程中创建并启动一个完整的会话对象。该函数完成以下工作：

1. 从统计模块获取活跃会话计数器（`shared_ptr` 包装），设置关闭回调用于会话结束时递减计数
2. 创建可靠传输层（`make_reliable`）封装底层 socket
3. 通过 `make_session()` 工厂函数创建会话对象
4. 记录会话开启（`session_open()`）
5. 设置会话关闭回调（递减活跃会话计数器）
6. 判断是否启用认证：检查 `config.agent.auth.users` 是否非空
7. 设置账户目录（认证禁用时传入 `nullptr`）
8. 设置凭证验证器（`account::contains` 校验凭证）
9. 设置出站代理（通过 worker 的 `outbound::direct` 实例）
10. 调用 `session::start()` 启动会话处理流程

如果启动过程中发生异常，回滚会话计数（`session_close()`）后重新抛出。

#### 签名
```cpp
void start(server_context &server, worker_context &worker, stats::state &metrics, tcp::socket socket);
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `server` | `server_context &` | 服务端全局上下文，包含配置和账户存储 |
| `worker` | `worker_context &` | 当前 worker 的线程局部上下文 |
| `metrics` | `stats::state &` | 当前 worker 的统计状态对象 |
| `socket` | `tcp::socket` | 已连接的 TCP socket，将被移动到会话中 |

#### 返回值
无。

#### 调用（向下）
- `metrics.session_counter()` — 获取活跃会话计数器的共享指针
- `metrics.session_open()` — 递增活跃会话计数
- `channel::transport::make_reliable()` — 将原始 socket 封装为可靠传输层
- `session::make_session()` — 创建会话对象
- `shared_session->set_on_closed()` — 设置关闭回调（递减计数器）
- `shared_session->set_account_directory()` — 设置账户目录
- `shared_session->set_credential_verifier()` — 设置凭证验证器
- `shared_session->set_outbound_proxy()` — 设置出站代理
- `shared_session->start()` — 启动会话处理流程
- `account::contains()` — 凭证验证

#### 被调用（向上）
- `launch::dispatch()` — 在 worker 线程中 socket 预配置完成后调用

#### 知识域
- [[agent/session/session|会话生命周期]] — 创建、配置、启动会话的完整流程
- [[agent/account/directory|账户认证]] — 凭证验证器和账户目录的设置
- [[agent/worker/stats|负载统计]] — 会话计数器的递增/递减管理
- [[channel/transport/reliable|可靠传输层]] — 封装原始 socket 为传输层抽象

---

## 函数: dispatch()

#### 功能说明
将 socket 分发到目标 worker 的事件循环。该函数实现了跨线程连接分发机制：主线程 acceptor 接收新连接后调用此函数，将 socket 投递到指定 worker 的 `io_context` 中异步执行。

分发过程：
1. 递增待处理计数（`handoff_push`），让负载均衡器感知排队压力
2. 通过 `net::post()` 将处理任务投递到目标 worker 的 `io_context`
3. 在 worker 线程中执行：递减待处理计数（`handoff_pop`）、迁移 socket executor、调用 `prime()` 预配置、调用 `start()` 启动会话
4. 所有异常被捕获并记录日志，不会传播到事件循环外部

#### 签名
```cpp
void dispatch(net::io_context &ioc, server_context &server, worker_context &worker, stats::state &metrics, tcp::socket socket);
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `ioc` | `net::io_context &` | 目标 worker 的 io_context |
| `server` | `server_context &` | 服务端全局上下文 |
| `worker` | `worker_context &` | 当前 worker 的线程局部上下文 |
| `metrics` | `stats::state &` | 当前 worker 的统计状态对象 |
| `socket` | `tcp::socket` | 已连接的 TCP socket，将被移动到投递任务中 |

#### 返回值
无。

#### 调用（向下）
- `metrics.handoff_push()` — 递增待处理连接计数
- `net::post(ioc, ...)` — 跨线程投递任务到目标 io_context
- `metrics.handoff_pop()` — 递减待处理连接计数
- `migrate_executor()` — 将 socket executor 迁移到目标 io_context
- `prime()` — 预配置 socket 参数（TCP_NODELAY、缓冲区大小）
- `start()` — 启动新会话

#### 被调用（向上）
- [[agent/worker/worker|worker::dispatch_socket]] — worker 接收 balancer 分发的 socket 后调用

#### 知识域
- [[ref/programming/boost-asio|Boost.Asio 跨线程投递]] — `net::post()` 实现线程安全的任务投递
- [[agent/worker/stats|负载统计]] — handoff 计数反映排队压力
- [[agent/front/balancer|负载均衡]] — dispatch 是连接分发的最后一环

---

## 调用链总览

```
[[agent/front/balancer|balancer::dispatch]] → [[agent/worker/worker|worker::dispatch_socket]]
    → launch::dispatch()
        → metrics.handoff_push()
        → net::post(ioc, lambda)
            → metrics.handoff_pop()
            → migrate_executor()  (socket executor 迁移)
            → prime()             (TCP 参数预配置)
            → start()             (会话创建与启动)
                → make_reliable()        (socket → 传输层)
                → [[agent/session/session|make_session()]]  (创建会话)
                → session::set_on_closed()    (关闭回调)
                → session::set_account_directory()  (账户目录)
                → session::set_credential_verifier() (凭证验证)
                → session::set_outbound_proxy()      (出站代理)
                → session::start()                   (启动会话)
                    → session::diversion() → [[recognition/recognition|recognize]] → [[agent/dispatch/table|dispatch]]
```

## 知识域

- [[ref/programming/boost-asio|Boost.Asio 跨线程模型]] — `post()` 投递 + executor 迁移实现线程安全分发
- [[agent/session/session|会话生命周期]] — 创建、配置、启动会话的完整流程
- [[agent/worker/worker|线程模型]] — 每线程一个 io_context 的无锁设计
- [[agent/worker/stats|负载统计]] — handoff 计数和会话计数的管理
- [[channel/transport/reliable|可靠传输层]] — socket 到传输层抽象的封装
- [[agent/account/directory|账户认证]] — 凭证验证器的设置逻辑
