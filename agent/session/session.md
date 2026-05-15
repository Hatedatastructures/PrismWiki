---
title: "session.hpp — 连接会话编排模块"
source: "include/prism/agent/session/session.hpp"
module: "agent"
type: api
tags: [agent, session, 会话, 协议检测, 生命周期]
created: 2026-05-15
updated: 2026-05-15
related:
  - agent/worker/launch
  - agent/worker/worker
  - agent/dispatch/table
  - recognition/recognition
  - agent/context
  - memory/pool
  - channel/transport/reliable
---

# session.hpp

> 源码: `include/prism/agent/session/session.hpp` + `src/prism/agent/session/session.cpp`
> 模块: [[agent|Agent]] / session

## 概述

定义会话管理核心组件，负责单个入站连接的完整生命周期。会话对象持有入站传输层，执行协议检测后分派到对应管道入口，若无匹配的专用处理路径则回退到原始透传模式。会话通过 `shared_from_this` 实现异步生命周期管理，确保协程执行期间对象不会被提前销毁。

生命周期管理采用"先停、再收"模型：`close()` 只负责标记关闭状态、取消底层连接，不立即 reset 传输对象；资源释放在主处理协程退出后或析构时统一进行，避免异步操作访问已释放对象。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/context|context]] | session_context / server_context / worker_context 定义 |
| 依赖 | [[memory/pool|pool]] | frame_arena 帧内存池 |
| 依赖 | [[recognition/recognition|recognition]] | 协议识别（外层探测 + TLS 伪装方案识别） |
| 依赖 | [[agent/dispatch/table|dispatch]] | 编译期协议处理函数表 |
| 依赖 | [[pipeline/primitives|primitives]] | tunnel 原始透传 |
| 被依赖 | [[agent/worker/launch|launch]] | 创建并启动会话 |

## 命名空间

`psm::agent::session`

---

## 辅助: detail::generate_session_id()

### 功能说明
生成全局唯一的 64 位会话 ID。

### 签名
```cpp
[[nodiscard]] inline std::uint64_t generate_session_id() noexcept
```

### 参数
无。

### 返回值
| 类型 | 说明 |
|------|------|
| `std::uint64_t` | 原子递增的全局唯一 ID，从 1 开始 |

### 调用（向下）
无。

### 被调用（向上）
- `session::session()` 构造函数

### 知识域
- [[ref/programming/cpp-atomics|原子操作]] — 使用 `std::atomic` 保证线程安全的 ID 生成

---

## 结构体: session_params

### 概述
会话初始化参数集合，封装创建会话所需的所有外部依赖。采用结构体聚合参数可提高接口稳定性，便于后续扩展新参数而不破坏现有调用点。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `server_context &` | `server` | 服务器全局上下文引用 |
| `worker_context &` | `worker` | 工作线程上下文引用 |
| `shared_transmission` | `inbound` | 入站传输层所有权 |

> **生命周期要求**: `server` 和 `worker` 引用的生命周期必须长于会话。
> **所有权转移**: `inbound` 的所有权将转移到会话对象内部。

---

## 类: session

### 概述
代理连接会话管理器，是单个代理连接的完整生命周期管理者。从入站连接建立开始，经过协议检测、管道分派、数据转发，直到连接关闭结束。

### 类层次
继承 `std::enable_shared_from_this<session>`。

### 设计意图
- 会话对象必须通过 `std::shared_ptr` 管理，禁止在栈上创建
- 构造后应立即调用 `start()` 启动异步处理流程
- 协议检测采用预读探测方式，根据前 24 字节数据识别协议类型

### 枚举: state

```cpp
enum class state : std::uint8_t { active, closing, closed };
```

| 值 | 说明 |
|----|------|
| `active` | 活跃状态，正常处理中 |
| `closing` | 正在关闭，已取消底层连接 |
| `closed` | 已关闭，资源已释放 |

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `std::uint64_t` | `id_` | 会话唯一标识符 |
| `memory::frame_arena` | `frame_arena_` | 帧内存池 |
| `state` | `state_` | 会话状态（单线程 io_context，无需原子） |
| `std::function<void()>` | `on_closed_` | 关闭回调 |
| `session_context` | `ctx_` | 会话上下文，持有所有状态 |

---

### session::session()

#### 功能说明
构造会话对象，初始化所有核心组件。从参数中提取服务器上下文和工作线程上下文的引用，并接管入站传输层的所有权。通过 `detail::generate_session_id()` 生成全局唯一 ID。

#### 签名
```cpp
explicit session(session_params params);
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `params` | `session_params` | 会话参数集合，包含服务器上下文、工作线程上下文和入站传输层 |

#### 返回值
无（构造函数）。

#### 调用（向下）
- `detail::generate_session_id()` — 生成唯一会话 ID
- `session_context` 构造函数 — 初始化会话上下文

#### 被调用（向上）
- `make_session()` — 工厂函数内部调用

#### 知识域
- [[memory/pool|PMR 内存管理]] — frame_arena 为会话期间的临时分配提供帧竞技场
- [[agent/context|上下文体系]] — session_context 聚合了会话所需的所有运行时资源

---

### session::~session()

#### 功能说明
析构会话对象，自动关闭所有关联的传输层并释放资源。调用 `release_resources()` 确保传输层正确关闭。析构函数是幂等的，多次调用 `close()` 无副作用。

#### 签名
```cpp
~session();
```

#### 参数
无。

#### 返回值
无（析构函数）。

#### 调用（向下）
- `release_resources()` — 释放所有资源

#### 被调用（向上）
- `std::shared_ptr` 引用计数归零时自动调用

#### 知识域
- [[ref/programming/c++23-coroutines|生命周期管理]] — "先停、再收"模型确保异步安全

---

### session::start()

#### 功能说明
启动会话异步处理流程。构造处理协程并通过 `net::co_spawn` 在 worker 的 `io_context` 上执行。协程内部调用 `diversion()` 进行协议识别和分流。异常分为两类处理：`exception::deviant` 输出完整诊断信息，标准异常仅输出 `what()` 消息。无论正常结束还是异常，最终都会调用 `release_resources()` 释放资源。

#### 签名
```cpp
void start();
```

#### 参数
无。

#### 返回值
无。

#### 调用（向下）
- `shared_from_this()` — 获取自身共享指针，保活协程执行期间的对象生命周期
- `diversion()` — 协议识别与分流（核心流程）
- `release_resources()` — 异常或正常结束后释放资源
- `net::co_spawn()` — 在 worker 的 io_context 上启动协程

#### 被调用（向上）
- [[agent/worker/launch|launch::start()]] — 会话创建后立即调用

#### 知识域
- [[ref/programming/c++23-coroutines|协程生命周期]] — `shared_from_this()` 捕获确保协程执行期间对象不被销毁
- [[ref/programming/boost-asio|Boost.Asio]] — `co_spawn` + `detached` 模式启动独立协程
- [[exception/deviant|异常层次]] — 区分 deviant 和标准异常的诊断输出

---

### session::close()

#### 功能说明
关闭会话并释放资源。标记会话为关闭状态，取消底层连接。采用"先停、再收"模型：只负责标记关闭和取消连接，不立即 reset 传输对象；资源释放在主处理协程退出后或析构时统一进行。该方法是幂等的，多次调用无副作用。

#### 签名
```cpp
void close();
```

#### 参数
无。

#### 返回值
无。

#### 调用（向下）
- `active_stream_cancel()` — 取消当前活跃流的异步操作
- `ctx_.inbound->cancel()` — 取消入站传输层的异步操作
- `ctx_.outbound->cancel()` — 取消出站传输层的异步操作

#### 被调用（向上）
- 外部组件在需要提前终止会话时调用
- `release_resources()` 在状态不是 `active` 时跳过实际关闭

#### 知识域
- [[ref/programming/boost-asio|Boost.Asio 取消语义]] — `cancel()` 触发异步操作的 `operation_aborted` 错误
- [[ref/programming/c++23-coroutines|状态机模型]] — 三态 `active → closing → closed` 保证幂等

---

### session::set_outbound_proxy()

#### 功能说明
设置出站代理指针。设置后 pipeline 中的 dial/forward 调用将通过此代理建立上游连接。如果未设置，pipeline 回退到直接使用 router 的旧路径。

#### 签名
```cpp
void set_outbound_proxy(outbound::proxy *proxy) noexcept;
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `proxy` | `outbound::proxy *` | 出站代理指针（由 worker 拥有，非拥有指针） |

#### 返回值
无。

#### 调用（向下）
无（直接赋值成员变量 `ctx_.outbound_proxy`）。

#### 被调用（向上）
- [[agent/worker/launch|launch::start()]] — 启动会话前设置出站代理

#### 知识域
- [[outbound/proxy|出站代理抽象]] — 出站代理接口决定上游连接的建立方式

---

### session::set_credential_verifier()

#### 功能说明
设置用户凭证验证回调。该函数在需要验证用户身份时被调用（如 SOCKS5 或 HTTP 代理认证场景）。验证器接收用户凭证字符串视图，返回布尔值表示验证结果。如果未设置验证器，所有认证请求将失败。

#### 签名
```cpp
void set_credential_verifier(std::function<bool(std::string_view)> verifier);
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `verifier` | `std::function<bool(std::string_view)>` | 验证函数，输入用户凭证，返回验证结果 |

#### 返回值
无。

#### 调用（向下）
无（直接赋值成员变量 `ctx_.credential_verifier`）。

#### 被调用（向上）
- [[agent/worker/launch|launch::start()]] — 根据配置设置凭证验证器

#### 知识域
- [[agent/account/entry|账户管理]] — 凭证验证与账户注册表配合实现用户认证

---

### session::set_account_directory()

#### 功能说明
设置账户注册表指针，用于限制用户连接数，防止单用户滥用。账户注册表是线程安全的，可被多个会话并发访问。

#### 签名
```cpp
void set_account_directory(account::directory *account_directory) noexcept;
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `account_directory` | `account::directory *` | 账户运行时注册表指针 |

#### 返回值
无。

#### 调用（向下）
无（直接赋值成员变量 `ctx_.account_directory_ptr`）。

#### 被调用（向上）
- [[agent/worker/launch|launch::start()]] — 认证启用时设置账户目录

#### 知识域
- [[agent/account/directory|账户目录]] — 账户注册表管理用户配额和连接数限制

---

### session::set_on_closed()

#### 功能说明
设置会话关闭回调。回调在 `close()` 幂等执行成功后触发一次，可用于资源清理、统计更新或通知外部监听者。回调执行顺序在传输层关闭之后、会话对象销毁之前。

#### 签名
```cpp
void set_on_closed(std::function<void()> callback) noexcept;
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `callback` | `std::function<void()>` | 关闭回调函数 |

#### 返回值
无。

#### 调用（向下）
无（直接赋值成员变量 `on_closed_`）。

#### 被调用（向上）
- [[agent/worker/launch|launch::start()]] — 设置会话关闭时的统计计数器递减回调

#### 知识域
- [[agent/worker/stats|负载统计]] — 关闭回调用于递减活跃会话计数器

---

### session::id()

#### 功能说明
获取会话 ID。会话 ID 是全局唯一的 64 位整数，从 1 开始递增，用于日志追踪和问题定位。

#### 签名
```cpp
[[nodiscard]] std::uint64_t id() const noexcept;
```

#### 参数
无。

#### 返回值
| 类型 | 说明 |
|------|------|
| `std::uint64_t` | 会话唯一标识符 |

#### 调用（向下）
无。

#### 被调用（向上）
- 日志输出、调试诊断场景

#### 知识域
- 会话标识 — 原子递增的全局唯一 ID

---

### session::diversion()（私有）

#### 功能说明
协议分流处理，是会话的核心流程。首先调用 `recognition::recognize()` 执行完整识别流程（外层探测 + TLS 伪装方案识别），获取识别结果后更新入站传输层（识别过程可能替换传输层，如 TLS 解包），然后通过 `dispatch::dispatch()` 分发到对应协议处理器。如果识别失败或无匹配处理器，记录警告日志并关闭会话。

#### 签名
```cpp
auto diversion() -> net::awaitable<void>;
```

#### 参数
无。

#### 返回值
| 类型 | 说明 |
|------|------|
| `net::awaitable<void>` | 协程任务，co_await 后执行协议分流 |

#### 调用（向下）
- `recognition::recognize()` — 执行完整协议识别流程（外层探测 + TLS 伪装方案识别）
- `dispatch::dispatch()` — 根据识别结果分发到对应协议处理器（编译期函数表）
- `protocol::to_string_view()` — 将协议类型转为字符串用于日志

#### 被调用（向上）
- `session::start()` — 通过 `co_spawn` 启动的协程内部调用

#### 知识域
- [[recognition/recognition|协议识别]] — 三阶段流水线：probe → identify → scheme execution
- [[agent/dispatch/table|协议分发]] — 编译期常量数组，零虚函数、零动态分配
- [[pipeline/primitives|管道原语]] — tunnel 原始透传作为 fallback 路径

---

### session::release_resources()（私有）

#### 功能说明
释放所有资源。关闭并释放入站和出站传输层对象，触发关闭回调。该方法只在确定没有异步操作运行时调用（协程退出后或析构时），确保安全释放。幂等的，多次调用无副作用。

#### 签名
```cpp
void release_resources() noexcept;
```

#### 参数
无。

#### 返回值
无。

#### 调用（向下）
- `active_stream_close()` — 关闭活跃流（如存在）
- `ctx_.inbound->close()` + `reset()` — 关闭并释放入站传输层
- `ctx_.outbound->shutdown_write()` + `close()` + `reset()` — 关闭并释放出站传输层
- `on_closed_()` — 触发关闭回调（移出后调用，防止重入）

#### 被调用（向上）
- `session::~session()` — 析构时调用
- `session::start()` 协程正常结束或异常退出后调用

#### 知识域
- [[ref/programming/c++23-coroutines|资源释放时机]] — 协程退出后统一释放，避免异步操作访问已释放对象
- [[channel/transport/reliable|可靠传输层]] — close() 和 shutdown_write() 的语义

---

## 工厂函数: make_session()

#### 功能说明
创建会话对象的工厂函数。返回 `std::shared_ptr` 管理的会话实例，满足 `enable_shared_from_this` 的前提条件。工厂函数内部调用 `std::make_shared`，相比直接 `new` 具有更好的内存局部性（控制块与对象连续分配）。

#### 签名
```cpp
std::shared_ptr<session> make_session(session_params &&params);
```

#### 参数
| 参数 | 类型 | 说明 |
|------|------|------|
| `params` | `session_params &&` | 会话参数集合，右值引用，所有权转移 |

#### 返回值
| 类型 | 说明 |
|------|------|
| `std::shared_ptr<session>` | 新创建的会话对象共享指针 |

#### 调用（向下）
- `session::session()` — 构造会话对象
- `std::make_shared<session>()` — 分配内存并构造

#### 被调用（向上）
- [[agent/worker/launch|launch::start()]] — 会话创建的唯一入口

#### 知识域
- [[ref/programming/cpp-smart-pointers|共享指针]] — `make_shared` 保证 `enable_shared_from_this` 可用
- [[agent/worker/launch|会话启动]] — 工厂函数是会话创建链路的中间环节

---

## 调用链总览

```
[[agent/worker/launch|launch::start]] → make_session() → session::start() → session::diversion()
    → [[recognition/recognition|recognition::recognize]] → [[agent/dispatch/table|dispatch::dispatch]]
    → [[pipeline/protocols|protocol handler]] 或 [[pipeline/primitives|tunnel]]

session::~session() → release_resources()
session::close() → cancel (inbound/outbound)
```

## 知识域

- [[ref/programming/c++23-coroutines|协程生命周期管理]] — shared_from_this 保活 + "先停、再收"释放模型
- [[recognition/recognition|协议识别]] — 三阶段流水线：probe → identify → scheme execution
- [[agent/dispatch/table|协议分发]] — 编译期 handler_table，零虚函数
- [[agent/context|上下文体系]] — session_context 聚合会话全部运行时资源
- [[memory/pool|帧内存池]] — frame_arena 为会话期间的 PMR 分配提供竞技场
