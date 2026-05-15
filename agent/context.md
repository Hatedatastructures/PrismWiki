---
title: "context.hpp — Agent 运行时上下文类型定义"
source: "include/prism/agent/context.hpp"
module: "agent"
type: api
tags: [agent, context, 上下文, 服务器, 工作线程, 会话]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[agent/worker/worker|Worker 核心]]"
  - "[[agent/session/session|会话管理]]"
  - "[[agent/account/directory|账户目录]]"
  - "[[ref/memory/pmr|PMR 内存管理]]"
---

# context.hpp

> 源码: `include/prism/agent/context.hpp`
> 模块: [[agent|Agent]]

## 概述

声明代理服务运行时的上下文结构，包括服务器上下文、工作线程上下文和会话上下文。这些上下文结构贯穿整个请求处理生命周期，为各层组件提供配置、资源和状态访问入口。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[agent/config\|config]] | agent::config 证书配置 |
| 依赖 | [[config\|全局配置]] | 顶层 psm::config |
| 依赖 | [[resolve/router\|router]] | DNS 路由器 |
| 依赖 | [[agent/account/entry\|entry]] | 账户条目与租约 |
| 依赖 | [[memory/pool\|pool]] | 帧内存池 |
| 依赖 | [[channel/transport/transmission\|transmission]] | 传输层抽象 |
| 依赖 | [[outbound/proxy\|proxy]] | 出站代理接口 |
| 被依赖 | [[agent/session/session\|session]] | 会话管理器持有 session_context |
| 被依赖 | [[agent/worker/worker\|worker]] | Worker 创建 server_context 和 worker_context |
| 被依赖 | [[pipeline/primitives\|primitives]] | 管道原语使用 session_context |
| 被依赖 | [[agent/dispatch/table\|table]] | 分发时传递 session_context |

## 命名空间

`psm::agent`

---

## 结构体: server_context

### 概述
服务器全局上下文，聚合服务器级别的共享资源，包括配置、SSL 上下文和账户注册表。

### 设计意图
该结构在服务器启动时创建，被所有工作线程共享。配置对象通过 `shared_ptr` 管理，支持原子交换实现配置热加载。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `std::atomic<std::shared_ptr<const psm::config>>` | `cfg` | 配置对象（可原子交换） |
| `std::shared_ptr<ssl::context>` | `ssl_ctx` | SSL 上下文 |
| `std::shared_ptr<account::directory>` | `account_store` | 账户注册表 |

---

### config()

**功能说明**: 无锁读取当前配置对象。

**签名**:
```cpp
[[nodiscard]] auto config() const -> const psm::config &;
```

**参数**: 无

**返回值**: `const psm::config &` 配置对象的常量引用

**调用（向下）**: `cfg.load()` 原子加载 shared_ptr 并解引用

**被调用（向上）**: 整个请求处理链路中需要读取配置的各层组件，包括 [[agent/session/session\|session]]、[[agent/dispatch/table\|dispatch]]、[[pipeline/protocols\|各协议处理器]]

**涉及的知识域**: [[ref/programming/c++23-coroutines\|生命周期管理]]

---

### swap_config()

**功能说明**: 原子交换配置对象，用于配置热加载。

**签名**:
```cpp
void swap_config(std::shared_ptr<const psm::config> new_cfg);
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `new_cfg` | `std::shared_ptr<const psm::config>` | 新配置对象 |

**返回值**: 无

**调用（向下）**: `cfg.store(std::move(new_cfg))` 原子存储

**被调用（向上）**: 外部管理代码在热加载配置时调用

**涉及的知识域**: [[ref/programming/c++23-coroutines\|生命周期管理]]

---

## 结构体: worker_context

### 概述
工作线程上下文，封装单个工作线程的独立资源。每个工作线程拥有独立的 `worker_context` 实例，实现线程间的资源隔离和避免锁竞争。

### 设计意图
io_context 的生命周期由工作线程管理。内存池资源指针用于 PMR 分配。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `net::io_context &` | `io_context` | I/O 上下文引用 |
| `resolve::router &` | `router` | 路由器引用 |
| `memory::resource_pointer` | `memory_pool` | 内存池资源指针 |
| `outbound::proxy *` | `outbound` | 出站代理指针（由 worker 拥有） |

---

## 结构体: session_context

### 概述
会话上下文，聚合单个连接会话所需的所有资源和状态，是请求处理流程的核心数据结构。

### 设计意图
该结构在会话创建时初始化，随会话生命周期销毁。包含服务器上下文和工作线程上下文的引用、帧内存池、凭据验证器、缓冲区配置以及入站出站传输对象。帧内存池用于会话期间的临时分配，会话结束后自动回收。

### 生命周期
- 不可拷贝（删除拷贝构造和拷贝赋值）
- 支持移动构造（默认）
- 不可移动赋值（删除）

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `std::uint64_t` | `session_id` | 会话唯一标识符 |
| `const server_context &` | `server` | 服务器上下文常量引用 |
| `worker_context &` | `worker` | 工作线程上下文引用 |
| `memory::frame_arena &` | `frame_arena` | 帧内存池引用 |
| `std::function<bool(std::string_view)>` | `credential_verifier` | 凭据验证函数 |
| `account::directory *` | `account_directory_ptr` | 账户注册表指针（可能为空） |
| `std::uint32_t` | `buffer_size` | 数据传输缓冲区大小（字节） |
| `shared_transmission` | `inbound` | 入站传输对象 |
| `shared_transmission` | `outbound` | 出站传输对象 |
| `outbound::proxy *` | `outbound_proxy` | 出站代理指针 |
| `account::lease` | `account_lease` | 账户连接租约 |
| `std::function<void()>` | `active_stream_cancel` | 活跃流取消回调 |
| `std::function<void()>` | `active_stream_close` | 活跃流关闭回调 |

---

### 构造函数

**功能说明**: 初始化会话上下文，绑定服务器和工作线程上下文引用，移动入站传输对象。

**签名**:
```cpp
session_context(const std::uint64_t sid, const server_context &srv, worker_context &w,
                memory::frame_arena &arena, account::directory *dir, std::function<bool(std::string_view)> verifier,
                const std::uint32_t buf_size, shared_transmission in);
```

**参数表格**:

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `sid` | `const std::uint64_t` | 会话唯一标识符 |
| `srv` | `const server_context &` | 服务器上下文常量引用 |
| `w` | `worker_context &` | 工作线程上下文引用 |
| `arena` | `memory::frame_arena &` | 帧内存池引用 |
| `dir` | `account::directory *` | 账户注册表指针，可为空 |
| `verifier` | `std::function<bool(std::string_view)>` | 凭据验证函数 |
| `buf_size` | `const std::uint32_t` | 数据传输缓冲区大小（字节） |
| `in` | `shared_transmission` | 入站传输对象 |

**返回值**: 无（构造函数）

**调用（向下）**: `std::move(verifier)`、`std::move(in)` 移动语义初始化成员

**被调用（向上）**: [[agent/session/session\|session 构造函数]] 内部创建 `session_context`

**涉及的知识域**: [[ref/programming/c++23-coroutines\|生命周期管理]]

---

## 知识域

- [[ref/programming/c++23-coroutines\|生命周期管理]]
- [[ref/memory/pmr\|PMR 内存管理]]
- [[ref/network/connection-pool\|连接配额管理]]
