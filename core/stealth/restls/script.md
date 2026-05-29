---
layer: core
source: include/prism/stealth/restls/script.hpp
title: Restls Script Engine
created: 2026-05-25
updated: 2026-05-27
tags: [stealth, restls, script, traffic-shaping, padding]
---
# Restls Script Engine

> 源码位置: include/prism/stealth/restls/script.hpp + src/prism/stealth/restls/script.cpp

## 设计决策（WHY）

### 为什么使用 script 控制流量

TLS 流量指纹分析可以通过帧大小分布和发送时序识别代理流量。Restls script 允许管理员精确控制每帧的发送大小和等待时间，模拟特定 HTTPS 应用的流量模式（如浏览器加载网页时的特征性流量）。

### 为什么 script_line 分为 send/wait/recv 三种类型

- **send**（N?T）：发送指定大小数据后等待
- **wait**（N~T）：等待指定时间后发送  
- **recv**（<N）：等待客户端发送 N 个数据包

这三种操作覆盖了流量控制的所有模式。send 控制服务端主动发送，wait 控制发送时机，recv 控制与客户端交互的节奏。

### 为什么 script_engine 需要 PMR 分配

script_engine 在热路径中使用（每次传输都可能执行 script 指令），使用 PMR 分配避免堆分配开销。script_line 数组使用帧竞技场分配。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| script 语法必须合法 | 解析器 | 非法语法导致配置加载失败 |
| send 操作的字节数不能超过 max_plaintext | 帧限制 | 16384 字节 |
| script 执行是同步的 | 设计 | 不涉及 I/O，仅控制缓冲区 |
| 空 script 等于无流量控制 | 默认行为 | 直接透传 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| script 语法错误 | 非法字符 | 配置加载失败 |
| 字节数超出限制 | N > 16384 | 运行时错误 |
| PMR 分配失败 | 内存耗尽 | 异常 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| script_engine -> config | 依赖 | restls_script 字符串输入 |
| handshake -> script_engine | 创建 | 握手中解析并创建 engine |
| transport -> script_engine | 使用 | 传输时执行 script 控制 |
