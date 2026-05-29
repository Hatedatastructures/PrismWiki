---
layer: core
source: include/prism/multiplex/duct.hpp
title: multiplex::duct — TCP 双向转发管道
module: multiplex
tags: [multiplex, duct, tcp, bidirectional, half-close]
updated: 2026-05-27
---

# multiplex::duct — TCP 双向转发管道

协议无关的双向 TCP 转发管道。每条 duct 绑定一个已连接的 target 传输层，提供 mux 帧到 target 的透明双向转发。

## 数据流向

| 方向 | 路径 | 说明 |
|------|------|------|
| 上传 | mux PSH → `on_mux_data` → target async_write | 客户端→目标 |
| 下载 | target async_read → `target_readloop` → core::send_data | 目标→客户端 |

## 核心接口

| 方法 | 说明 |
|------|------|
| `duct(config, owner, target)` | 构造，绑定 target 传输层 |
| `start()` | co_spawn 启动 target_readloop + target_writeloop |
| `on_mux_data(data)` | 接收 mux 数据写入 target |
| `close()` | 幂等关闭，从 core 的 ducts_ 移除 |
| `stream_id()` | 流标识符 |

## 设计决策

### 为什么 duct 的 owner_ 用 weak_ptr？

core 的 `ducts_` 持有 duct 的 shared_ptr。如果 duct 再持有 core 的 shared_ptr，形成循环引用。weak_ptr 打破循环：core 析构后 `lock()` 返回空，duct 的 readloop 检测到后安全退出。

**后果**: 每次 send_data/send_fin 前需要 `owner_.lock()` 检查。

### 为什么支持半关闭？

TCP 半关闭允许一方停止发送但仍接收。mux FIN 对应 TCP 半关闭：target EOF 后 duct 发送 FIN 给客户端，但客户端方向仍可发送数据直到客户端也发 FIN。双向都关闭后才完全关闭 duct。

**后果**: 需要分别跟踪 `mux_closed_` 和 `target_closed_` 状态。

### 为什么用 concurrent_channel 而非互斥队列？

duct 的 `on_mux_data` 由 craft 的 dispatch_push 通过 co_spawn 非阻塞调用。多个 mux 帧可能同时到达。`concurrent_channel` 提供无锁的多生产者单消费者队列，target_writeloop 作为消费者串行写入 target。

**后果**: 不需要互斥锁，符合 Prism 的纯协程架构。

## 约束

### 半关闭后 mux 端停止读取 target

**类型**: 生命周期

**规则**: `mux_closed_` 为 true 后 `target_readloop` 立即退出。yamux 客户端 FIN 后不再发 WindowUpdate，继续发送会导致窗口耗尽。

**违反后果**: 窗口耗尽后 send_data 无限等待（yamux）。

**源码依据**: `duct.cpp:199-201`

### read_size_ 受帧最大载荷限制

**类型**: 资源上限

**规则**: `read_size_ = min(buffer_size, max_frame_payload)`，确保单次读取不超过 mux 帧最大载荷 65535 字节。

**违反后果**: 超过帧载荷限制的数据无法编码为合法 mux 帧。

## 故障场景

| 场景 | 触发条件 | 处理 | 恢复 |
|------|----------|------|------|
| target 写入失败 | 目标关闭/超时 | target_writeloop → close() → mux FIN | 客户端新建流 |
| target EOF | 目标主动关闭 | target_readloop 退出 → send_fin → 半关闭 | 正常半关闭 |
| core 销毁 | core::close() | owner_.lock() 返回空 → duct 安全退出 | 无需恢复 |

## 引用关系

### 依赖

| 模块 | 用途 |
|------|------|
| [[core/multiplex/core\|core]] | owner_ 弱引用，send_data/send_fin |
| [[core/channel/transport/transmission\|transmission]] | target 传输层 |

### 被引用

| 模块 | 使用方式 |
|------|----------|
| [[core/multiplex/core\|core]] | ducts_ 映射管理 duct 生命周期 |
| smux::craft / yamux::craft | dispatch_data → on_mux_data |
