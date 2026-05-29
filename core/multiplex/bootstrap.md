---
layer: core
source: include/prism/multiplex/bootstrap.hpp
title: multiplex::bootstrap - 多路复用会话引导
tags: [multiplex, bootstrap, sing-mux, negotiate]
updated: 2026-05-28
---

# multiplex::bootstrap - 多路复用会话引导

## 源码位置

`include/prism/multiplex/bootstrap.hpp`

## 概述

`bootstrap` 函数是多路复用会话的统一入口，完成 sing-mux 协议协商后根据客户端选择的协议类型创建对应的 [[core/multiplex/core|core]] 子类实例。

## sing-mux 协商格式

### 基本格式（Version==0）

```
[Version 1B][Protocol 1B]
```

### 扩展格式（Version>0）

```
[Version 1B][Protocol 1B][PaddingLen 2B BE][Padding N bytes]
```

Padding 用途：对齐协商头到特定字节边界，避免流量分析识别 mux 协议。

### Protocol 字段含义

| Protocol 值 | 协议类型 |
|-------------|----------|
| 0 | [[core/multiplex/smux/craft|smux]] |
| 1 | [[core/multiplex/yamux/craft|yamux]] |

## 接口表

### bootstrap_context 结构

聚合 bootstrap 所需的所有参数，避免函数参数超过 3 个。调用者通过聚合初始化填充。

| 字段 | 类型 | 说明 |
|------|------|------|
| transport | shared_transmission | 已建立的传输层连接 |
| router | connect::router& | 路由器引用，用于解析地址并连接目标 |
| cfg | const config& | 多路复用配置 |
| mr | memory::resource_pointer | PMR 内存资源，为空时使用默认资源 |
| traffic | traffic_state* | per-worker 流量统计指针 |
| proto | protocol_type | 归属的外层协议类型 |

### bootstrap 函数

| 项目 | 说明 |
|------|------|
| 签名 | `auto bootstrap(bootstrap_context ctx) -> net::awaitable<shared_ptr<core>>` |
| 成功 | mux 会话实例的共享指针（指向具体子类） |
| 失败 | nullptr |

## 工作流程

1. 读取 sing-mux 协商头 `[Version][Protocol]`
2. Version>0 时读取 PaddingLen (2B 大端) 并丢弃 Padding
3. switch (Protocol)：0 创建 smux::craft，1 创建 yamux::craft，default 返回 nullptr
4. craft 构造函数持有 transport、绑定 router、应用 config、分配内存资源
5. 调用 core->start() 设置 active 标志并 co_spawn run() 协程

## 协议对比

| 特性 | smux (Protocol=0) | yamux (Protocol=1) |
|------|-------------------|-------------------|
| 来源 | xtaci/smux v1 + sing-mux | Hashicorp/yamux |
| 帧大小 | 固定头部 8 字节 | 固定头部 12 字节 |
| 流控 | 无窗口机制 | 增量窗口更新 |
| 心跳 | NOP 帧不回复 | Ping/Pong 机制 |
| 性能 | 低延迟，适合小帧 | 高吞吐，适合大帧 |

## 设计决策

### 为什么协商头只有 2 字节最小长度？

**问题**: 协商开销应尽可能小，因为它在 mux 帧数据之前发送，影响首流建立延迟。

**选择**: Version==0 时仅 `[Version 1B][Protocol 1B]` 共 2 字节。Version>0 时扩展 padding 用于流量混淆。

**后果**: 最小协商头 2 字节，比 HTTP/2 的 PRI + SETTINGS 帧小得多。Version 字段为未来协议扩展预留了空间。

### 为什么 bootstrap_context 使用聚合结构体？

**问题**: 原始 bootstrap 函数有 5 个参数（transport, router, cfg, mr, traffic），违反 Rule 1（不超过 3 个参数）。

**选择**: 引入 `bootstrap_context` 聚合所有参数，函数签名降为 1 个参数。

**后果**: 调用方通过聚合初始化填充，更清晰且易于扩展新参数。源码依据：`bootstrap.hpp:48-56`。

### 为什么协议类型由客户端决定？

**问题**: 服务端支持多种 mux 协议时，需要确定使用哪一种。

**选择**: 服务端不主动选择，完全由客户端协商头中的 Protocol 字段决定。`multiplex::config` 中的 smux 和 yamux 子配置仅用于已选定协议的行为参数。

**后果**: 一旦选定，整个会话生命周期内不可切换。Protocol 值非法时返回 nullptr。

## 兼容性

- Version=0 的客户端可以被 Version>0 的服务端正确解析
- Version>0 的客户端向 Version=0 服务端发送 padding 时，服务端可能将 padding 误认为协议数据（不兼容）

## 约束

### 协商期间 transport 不可用

- **类型**: 生命周期
- **规则**: bootstrap() 内部读取协商头期间，transport 的所有权尚未移交 craft。读取失败时 transport 被丢弃。
- **违反后果**: 无（bootstrap 返回 nullptr，调用方清理）
- **源码依据**: `bootstrap.cpp:87-92`

## 故障场景

### 协商头读取不完整

- **触发条件**: 客户端发送协商头前断开连接，或网络中断
- **传播路径**: `transport::async_read` 返回 n < 2 → 返回 `connection_reset` 错误码 → `bootstrap` 返回 nullptr
- **外部表现**: Agent Session 收到 nullptr，按普通非 mux 连接处理或关闭
- **恢复机制**: 客户端需重新连接
- **日志关键字**: `[Mux.Bootstrap] sing-mux negotiate failed`

### Padding 读取失败

- **触发条件**: Version>0 但 padding 数据读取时连接中断
- **传播路径**: `transport::async_read` 返回错误 → bootstrap 返回 nullptr
- **恢复机制**: 同上
- **日志关键字**: `[Mux.Bootstrap] sing-mux negotiate failed`

### craft 构造异常

- **触发条件**: 内存分配失败（PMR 资源耗尽）
- **传播路径**: `std::make_shared` 抛出 `std::bad_alloc` → catch 块捕获 → 返回 nullptr
- **恢复机制**: 客户端重试
- **日志关键字**: `[Mux.Bootstrap] create_session exception`

## 关联文档

- [[core/multiplex/core|core]] - 多路复用核心抽象基类
- [[core/multiplex/config|config]] - 多路复用配置
- [[core/multiplex/smux/craft|smux::craft]] - smux 协议实现
- [[core/multiplex/yamux/craft|yamux::craft]] - yamux 协议实现
