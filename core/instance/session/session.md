---
layer: core
source: include/prism/instance/session/session.hpp
title: session 模块
created: 2026-05-17
updated: 2026-05-28
tags: [instance, session, lifecycle, protocol-detect, diversion]
---

# session 模块

> 源码: `include/prism/instance/session/session.hpp`

## 模块职责

连接会话编排模块，负责单个入站连接的完整生命周期管理。会话对象持有入站传输层，执行协议检测后分派到对应管道入口，若无匹配的专用处理路径则回退到原始透传模式。通过 `shared_from_this` 实现异步生命周期管理，确保协程执行期间对象不会被提前销毁。

## 主要组件

### 全局会话 ID 生成

`detail::sid_counter` 全局原子计数器，`detail::next_sid()` 通过单次原子递增生成唯一会话 ID。

### session_params

会话初始化参数集合，封装创建会话所需的所有外部依赖。

| 成员 | 类型 | 说明 |
|------|------|------|
| `server` | `context::server &` | 服务器全局上下文引用 |
| `worker` | `context::worker &` | 工作线程上下文引用 |
| `inbound` | `shared_transmission` | 入站传输层所有权 |

### session 类

| 方法 | 说明 |
|------|------|
| `session(params)` | 构造函数，初始化所有核心组件 |
| `~session()` noexcept | 析构函数，自动关闭所有关联传输层 |
| `start()` | 启动异步处理流程（协议检测 + 转发） |
| `close()` | 关闭会话并释放资源（幂等） |
| `set_outbound_proxy(proxy)` | 设置出站代理 |
| `set_credential_verifier(verifier)` | 设置用户凭证验证回调 |
| `set_account_directory(directory)` | 设置账户注册表指针 |
| `set_on_closed(callback)` | 设置会话关闭回调 |
| `id()` | 获取会话唯一标识符 |

#### 私有方法

| 方法 | 说明 |
|------|------|
| `diversion()` | 协议分流处理，预读 24 字节识别协议 |
| `release_resources()` | 释放所有资源 |

#### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `id_` | `uint64_t` | 会话唯一标识符 |
| `frame_arena_` | `memory::frame_arena` | 帧内存池 |
| `state_` | `state` | 会话状态 |
| `on_closed_` | `function<void()>` | 关闭回调 |
| `handshake_deadline_` | `unique_ptr<steady_timer>` | 握手截止定时器 |
| `ctx_` | `context::session` | 会话上下文 |

### 工厂函数

`make_session(session_params &&)` 返回 `shared_ptr<session>`，确保会话对象始终通过共享指针管理，满足 `enable_shared_from_this` 的前提条件。

---

## 生命周期状态机

### 状态枚举

| 值 | 说明 |
|----|------|
| `active` | 活跃状态，正常处理中 |
| `closing` | 正在关闭，已取消底层连接 |
| `closed` | 已关闭，资源已释放 |

### 状态转换表

| 从 | 到 | 触发条件 | 幂等 |
|----|----|----------|------|
| active | closing | `close()` / 读异常 / EOF | 是 |
| closing | closed | 主处理协程退出 -> 析构函数 | 否 |
| active | closed | 析构函数直接触发（罕见） | 是 |

### 为什么用"先停、再收"模型

`close()` 只标记状态 + 取消底层 I/O，不立即释放资源。资源在主处理协程退出后或析构时统一释放。

**后果**: 避免协程恢复后访问已释放对象。

**约束**:
- type: 线程安全 — CAS 状态转换保证只有一次成功
- rule: 协程在 `co_await` 后检查 `state_ == closing`，防止使用已关闭的传输层

### 并发安全保证

`shared_from_this` 机制确保协程执行期间 session 对象存活。外部调用 `close()` 只标记状态，不持有强引用。协程是最后一个 `shared_ptr` 持有者，协程退出 -> 引用计数归零 -> `~session()` 执行。

---

## 协程绑定

`start()` 通过 `co_spawn` 将 `diversion()` 协程绑定到 worker 的 `io_context`：

- **executor**: `ctx_.worker_ctx.io_context`（所有异步操作调度到 worker 线程）
- **completion token**: `net::detached`（后台任务，不等待结果）
- **capture**: `shared_from_this()`（闭包持有强引用防止对象销毁）

### 为什么用 net::detached

Session 是独立后台任务，不需要等待其完成。生命周期由 `shared_ptr` + 状态机自动管理。使用 `use_awaitable` 会增加额外的 awaitable 句柄管理复杂度。

---

## 协议检测与分流 (diversion)

### 流程

1. 预读 24 字节探针数据
2. 调用 `detect_from_transmission` 识别协议类型
3. 从全局注册表获取对应处理器，分发到管道
4. 管道接管 inbound 传输层，session.diversion() 协程退出

### 协议特征检测规则

| 协议 | 检测条件 |
|------|----------|
| HTTP 请求 | buf 前缀匹配 `GET `/`POST `/`PUT `/`DELE`/`HEAD`/`OPTI`/`CONN`/`PATC` |
| HTTP 响应 | buf 前缀匹配 `HTTP/` |
| TLS | `buf[0]==0x16 && buf[1]==0x03` |
| SOCKS5 | `buf[0]==0x05 && buf[1]>=1` |
| 未知 | 以上均不匹配，走原始透传 |

### 为什么预读 24 字节

HTTP 方法名最长 7 字节 (OPTIONS) + 空格 = 8 字节。TLS 记录头 5 字节。SOCKS5 版本 + 方法数 2 字节。24 字节覆盖所有特征且留有余量，通常一次网络读取即可获取（小于 TCP MSS）。

---

## 资源释放时序

LIFO 顺序释放，确保依赖关系安全：

| 阶段 | 动作 |
|------|------|
| Phase 1 | 传输层关闭：`inbound.cancel()` + `outbound.cancel()`，挂起 I/O 返回 `operation_aborted` |
| Phase 2 | 协程取消等待：diversion() 检测到 cancel 后自然退出 |
| Phase 3 | 账户租约释放：`account_lease` 析构归还连接配额 |
| Phase 4 | 帧内存池回收：`frame_arena` 析构归还 PMR 池 |
| Phase 5 | 回调触发：`on_closed_()` 通知 worker 更新统计 |

---

## 错误处理

| 阶段 | 错误类型 | 处理策略 |
|------|----------|----------|
| 预读探针 | 读超时 / 连接重置 | `close()`，连接关闭 |
| 预读探针 | EOF (0 字节) | 静默关闭 |
| 协议检测 | 探针不足 | 归入未知，走透传 |
| 管道创建 | 内存分配失败 | `close()` |
| 管道运行 | I/O 错误 | 管道内部捕获，触发 `close()` |

强异常安全保证：协程退出时所有局部变量自动析构，`shared_ptr` 归零时析构函数确保完全清理。

## 故障模式

### 无 idle timeout

Session 没有空闲超时。恶意连接可无限期保持，长期运行后 fd 累积导致 EMFILE。

**诊断**: `lsof -p <pid> | wc -l` 检查 fd 增长趋势。

## 相关文档

- [[core/instance/context|上下文模块]]
- [[core/instance/worker/worker|Worker 模块]]
- [[core/instance/worker/launch|启动模块]]
- [[core/instance/worker/stats|统计模块]]
