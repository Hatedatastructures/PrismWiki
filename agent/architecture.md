---
title: Agent 模块架构设计
created: 2026-05-13
updated: 2026-05-13
type: concept
tags: [agent, architecture, design]
related: [[agent/overview], [agent/detail], [dev/modules]]
---

# Agent 模块架构设计

Agent 模块采用分层架构，从监听到转发共 5 层，每层职责清晰，协程驱动全链路。

## 分层架构

```
┌─────────────────────────────────────────────────┐
│  Front Layer                                    │
│  listener（独立 io_context + 反压）              │
│  → balancer（加权评分 + 亲和性哈希）             │
├─────────────────────────────────────────────────┤
│  Worker Layer                                   │
│  worker（per-thread io_context + 连接池 + 路由） │
│  → launch（socket 预配置 + 会话创建）            │
├─────────────────────────────────────────────────┤
│  Session Layer                                  │
│  session（生命周期管理 + 协议分流）               │
├─────────────────────────────────────────────────┤
│  Dispatch Layer                                 │
│  handler_table（编译期 constexpr 函数表）        │
├─────────────────────────────────────────────────┤
│  Account Layer                                  │
│  directory（无锁 COW）+ entry（原子计数）         │
└─────────────────────────────────────────────────┘
```

## 启动流程

1. `memory::system::enable_global_pooling()` — 初始化全局内存池
2. `stealth::register_all_schemes()` — 注册所有 TLS 伪装方案
3. 加载配置文件（`loader::load()`）
4. 初始化日志系统（`trace::init()`）
5. 构建账户目录（`loader::build_account_directory()`）
6. 创建 worker 线程池（CPU 核心数 - 1）
7. 构建负载均衡器（`balancer`）
8. 启动监听器（`listener`）
9. 启动所有线程（worker 线程 + 监听线程）

## 协议处理流程

1. `listener` 接受连接 → 计算亲和性哈希
2. `balancer` 选择 worker → 分发 socket
3. `worker` 投递到 `io_context` → `launch` 创建会话
4. `session` 调用 `recognition::recognize()`：
   - **Probe**：预读 24 字节，检测 HTTP/SOCKS5/TLS/SS2022
   - **Identify**（仅 TLS）：读取 ClientHello → 特征分析 → 方案执行
   - SS2022 无正特征，排除法 fallback
5. `dispatch` 根据 `detected` 类型获取 handler
6. `handler` 调用 pipeline → 通过 `router` 建立上游连接
7. `tunnel` 执行双向透明转发

## 关键设计决策

### 独立 io_context 监听

listener 使用独立的 io_context，与 worker 隔离。避免监听器和工作线程相互影响，代价是增加一次跨线程 socket 转移。

### 编译期函数表替代虚函数

dispatch::table 使用 constexpr 数组，零虚函数、零动态分配、编译时可内联。数组索引对应 protocol_type 枚举值。

### shared_from_this 生命周期管理

session 使用 enable_shared_from_this 管理生命周期，确保协程异步执行期间对象不被提前销毁。采用"先停、再收"模型：close() 只标记关闭状态，资源释放在协程退出后统一进行。

### 加权评分负载均衡

综合考虑活跃会话数（60%）、待处理移交数（10%）、事件循环延迟（30%）。进入过载阈值（90%）高于退出阈值（80%），滞后机制避免状态抖动。所有工作线程均过载时触发全局反压。

### 三级上下文隔离

- `server_context`：全局共享，配置 + SSL 上下文 + 账户目录，支持原子交换热加载
- `worker_context`：线程独占，io_context + 路由器 + 内存池 + 出站代理
- `session_context`：会话独占，入站/出站传输 + 帧内存池 + 凭据验证器

### 无锁账户目录

account::directory 使用原子共享指针实现无锁读取和写时复制更新。所有修改操作复制整个映射表，适用于读多写少场景。lease RAII 封装自动管理连接计数。

## 相关页面

- [[agent]] — 模块概述
- [[agent]] — 详细设计
- [[dev/modules]] — Prism 全局模块结构
- [[dev/cpp23-coroutines]] — 协程技术
- [[dev/pmr-memory-pool]] — PMR 内存池
