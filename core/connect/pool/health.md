---
title: health
layer: core
source: include/prism/connect/pool/health.hpp
module: connect
tags:
  - health-check
  - socket
  - connection-pool
  - reuse
  - FIN-detection
created: 2026-05-23
updated: 2026-05-23
---

# health

Socket 健康检测模块，提供 TCP socket 健康状态检测功能，用于 [[core/connect/pool/pool|connection_pool]] 复用前的有效性验证。

## 概述

在将连接归还到连接池之前，需要检测 socket 是否仍然健康可用，避免将已关闭或错误的连接放入池中供后续复用。`health.hpp` 提供两种检测策略：

- **`health()`**：完整健康检测，返回详细的 `socket_state` 枚举
- **`healthy_fast()`**：快速健康检测，返回简单的 `bool`，适合热路径

检测维度包括三个层次：

| 维度 | 方法 | 检测内容 |
|------|------|----------|
| SO_ERROR | `getsockopt()` | socket 的待处理错误码 |
| available | `socket.available()` | 是否有待读取数据 |
| peek | `recv(MSG_PEEK)` | 对端是否已发送 FIN |

## socket_state 枚举

```cpp
enum class socket_state
{
    healthy,   // 健康，可安全复用
    has_data,  // 有待读数据，可能是延迟数据或 FIN
    fin,       // 对端已关闭，不可复用
    error,     // socket 发生错误，不可复用
    invalid    // socket 无效（未打开或已关闭）
};
```

### 状态分类

```
socket_state 决策树:

socket.is_open()?
    |-- No  --> invalid
    |
    v
getsockopt(SO_ERROR) != 0?
    |-- Yes --> error
    |
    v
available() > 0?
    |-- Yes --> has_data
    |-- No  --> [peek 检测]
                    |
                    +-- would_block/try_again --> healthy
                    +-- n == 0                --> fin
                    +-- 其他                  --> has_data
```

### 状态含义

| 状态 | 含义 | 可复用 | 说明 |
|------|------|--------|------|
| `healthy` | 无错误、无待读数据、无 FIN | 是 | 理想的复用候选 |
| `has_data` | 有待读数据 | 否 | 可能残留上一轮的脏数据 |
| `fin` | 对端已发 FIN | 否 | 连接半关闭，不可复用 |
| `error` | socket 有错误 | 否 | 连接异常，不可复用 |
| `invalid` | socket 未打开 | 否 | 无效连接 |

## health() 函数

```cpp
[[nodiscard]] socket_state health(const net::ip::tcp::socket &s);
```

完整的 socket 健康检测，依次检查三个维度，综合判断 socket 是否可安全复用。

### 检测流程

```
health(socket)
    |
    v
[1. socket.is_open()?]
    |-- No --> return invalid
    |
    v
[2. getsockopt(SO_ERROR)]
    |-- 失败或 error != 0 --> return error
    |
    v
[3. socket.available()]
    |-- 错误 --> return error
    |-- > 0 --> return has_data
    |-- == 0 --> [4. peek 检测]
                    |
                    v
                设置非阻塞模式
                    |
                    v
                recv(MSG_PEEK, 1 byte)
                    |
                    +-- would_block/try_again --> return healthy
                    |   (无数据可读，连接正常)
                    |
                    +-- n == 0 --> return fin
                    |   (对端已发 FIN)
                    |
                    +-- n > 0 --> return has_data
                    |   (有数据但 available 未报告)
                    |
                    +-- 其他错误 --> return error
```

## 关键实现细节

**SO_ERROR 检测**: 使用原生 `getsockopt()` 而非 Boost.Asio 封装，避免异常。跨平台处理：Windows 使用 `int len`，Linux 使用 `socklen_t len`。

**available 检测**: `available()` 返回内核接收缓冲区中可读字节数。`avail > 0` 表示有数据待读，可能是上一轮的残留数据。

**peek 检测**: 使用 `MSG_PEEK` 预读 1 字节，不从缓冲区移除。`would_block/try_again` 表示连接正常，`n == 0` 表示对端已发 FIN。保存并恢复非阻塞状态，防止改变 socket 的阻塞模式。

## healthy_fast() 函数

```cpp
[[nodiscard]] bool healthy_fast(const net::ip::tcp::socket &s);
```

快速 socket 健康检测。检查 SO_ERROR、available 和 FIN 状态，返回 `true` 表示健康可复用。

### 检测流程

```
healthy_fast(socket)
    |
    v
[1. socket.is_open()?]
    |-- No --> return false
    |
    v
[2. getsockopt(SO_ERROR)]
    |-- 失败或 error != 0 --> return false
    |
    v
[3. socket.available()]
    |-- 错误 --> return false
    |-- > 0 --> return false
    |   (有待读数据，不适合复用)
    |
    v
[4. available == 0, peek 检测 FIN]
    设置非阻塞模式
    recv(MSG_PEEK, 1 byte)
    恢复非阻塞模式
    |
    +-- would_block/try_again --> return true (健康)
    +-- n == 0                --> return false (FIN)
    +-- 其他                  --> return false (不健康)
```

### health() vs healthy_fast()

| 特性 | `health()` | `healthy_fast()` |
|------|-----------|------------------|
| 返回类型 | `socket_state` | `bool` |
| 信息量 | 详细（5 种状态） | 简单（健康/不健康） |
| 适用场景 | 调试、诊断 | 热路径复用检测 |
| 性能 | 略慢 | 略快 |
| `has_data` 处理 | 返回 `has_data` | 返回 `false` |

**设计意图：**
- 连接池归还路径（热路径）使用 `healthy_fast()`，只需知道能否复用
- 调试和诊断场景使用 `health()`，获取详细的健康状态信息

## FIN 检测原理

TCP 连接的对端关闭时（调用 `close()` 或 `shutdown(WRITE)`），内核发送 FIN 包。接收端的 `recv()` 返回 0 表示收到 FIN。但 `available()` 不一定能检测到 FIN，因为 FIN 本身不产生"可用数据"。

```
available() 无法检测 FIN:

时间线:
  t0: 客户端调用 close()
  t1: 内核发送 FIN
  t2: 服务器收到 FIN
  t3: available() 仍然返回 0  <-- 无法检测
  t4: recv(MSG_PEEK) 返回 0   <-- 可以检测
```

因此，当 `available() == 0` 时必须通过 `recv(MSG_PEEK)` 进一步检测 FIN：

- `would_block / try_again`：连接正常（无数据也无 FIN）
- `n == 0`：收到 FIN（对端已关闭）
- `n > 0`：有数据（理论上 `available()` 应该 > 0，但存在竞态）

## 非阻塞状态保存

peek 检测需要临时将 socket 设为非阻塞模式。实现中保存并恢复原始非阻塞状态，避免影响后续异步操作。正常情况下连接池的 socket 已经是非阻塞模式（由 Boost.Asio 管理），此保护是防御性编程。

## 设计决策

### 为什么 FIN 检测需要 peek 而非 available？

TCP FIN 不产生"可用数据"——`available()` 返回 0 即使对端已发 FIN。只有 `recv(MSG_PEEK)` 返回 0 才能确认收到 FIN。因此当 `available() == 0` 时必须通过 peek 进一步检测。

### 为什么 healthy_fast 而非 health 用于热路径？

`healthy_fast()` 只需一次 bool 判断，而 `health()` 返回 5 种状态需要 switch 处理。连接池归还路径每秒可能执行数千次，省去分支开销有实际意义。

## 约束

### const_cast 安全性

**类型**: 线程安全

**规则**: `health()` 和 `healthy_fast()` 接受 `const socket &`，但 `available()` 和 `receive()` 需要非 const。内部使用 `const_cast`，保存并恢复非阻塞标志。

**违反后果**: 如果检测中途被中断（信号），非阻塞状态可能未正确恢复。实际中 Boost.Asio socket 总是非阻塞，风险极低。

**源码依据**: `health.hpp:57-58`

## 故障场景

### peek 检测阻塞

**触发条件**: socket 意外处于阻塞模式（非 Boost.Asio 管理），且 peek 时无数据可读也无 FIN。`receive()` 阻塞。

**传播路径**: `healthy_fast()` 阻塞 → `recycle()` 阻塞 → 连接归还延迟 → 池操作堆积。

**外部表现**: 连接池归还延迟，新连接获取变慢。

**恢复机制**: 防御性编程：peek 前显式设置非阻塞模式，即使恢复失败也不影响 socket 的异步使用。

**日志关键字**: 无直接日志（检测不产生日志）

### 跨平台 getsockopt 差异

**触发条件**: 在非 Windows/Linux 平台上编译，`socklen_t` 类型不存在。

**传播路径**: 编译失败。

**恢复机制**: 编译期 `#ifdef` 分支处理。

## 注意事项

1. **MSG_PEEK 副作用**: `MSG_PEEK` 不从缓冲区移除数据，对后续正常读取无影响
2. **性能**: `healthy_fast()` 涉及 3 次系统调用（`getsockopt` + `available` + `recv`），热路径开销可接受
3. **would_block vs try_again**: Windows 使用 `would_block`，Linux 使用 `try_again`，代码同时检查两者
4. **检测时机**: 健康检测只在归还和取出时执行，不会主动检测池中已缓存的空闲连接
