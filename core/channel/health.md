---
title: "health — Socket 健康检测"
layer: core
source: "I:/code/Prism/include/prism/connect/pool/health.hpp"
module: "channel"
type: api
tags: [channel, health, socket, 连接池, 健康检测]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/connect/pool/pool
  - core/transport/reliable
  - ref/network/tcp
---

# health — Socket 健康检测

> 源码: `I:/code/Prism/include/prism/connect/pool/health.hpp`
> 实现: `src/prism/channel/health.cpp`
> 模块: [[core/connect/overview|connect]]

## 模块职责

Socket 健康检测模块。提供 TCP socket 健康状态检测功能，用于连接池复用前的有效性验证。

在将连接归还到连接池之前，需要检测 socket 是否仍然健康可用，避免将已关闭或错误的连接放入池中供后续复用。

**检测维度**：
1. **SO_ERROR** — 获取 socket 的待处理错误码
2. **available** — 检查是否有待读取数据（可能是对端 FIN）
3. **peek** — 通过 `MSG_PEEK` 预读检测对端是否已发送 FIN

**健康状态分类**：
- `healthy` — 连接健康可安全复用
- `has_data` — 有待读数据，可能是延迟数据或 FIN
- `fin` — 对端已关闭，不可复用
- `error` — socket 发生错误，不可复用
- `invalid` — socket 无效（未打开或已关闭）

## 源码位置

| 文件 | 路径 |
|------|------|
| 头文件 | `I:/code/Prism/include/prism/connect/pool/health.hpp` |
| 实现 | `I:/code/Prism/src/prism/channel/health.cpp` |

## API

### 命名空间

`psm::channel`

### 枚举: socket_state

Socket 健康状态枚举。

```cpp
enum class socket_state {
    healthy,    // 健康，可安全复用
    has_data,   // 有待读数据（可能是延迟数据或 FIN）
    fin,        // 对端已关闭，不可复用
    error,      // socket 发生错误，不可复用
    invalid     // socket 无效（未打开或已关闭）
};
```

| 值 | 说明 |
|----|------|
| `healthy` | 健康，可安全复用 |
| `has_data` | 有待读数据（可能是延迟数据或 FIN） |
| `fin` | 对端已关闭，不可复用 |
| `error` | socket 发生错误，不可复用 |
| `invalid` | socket 无效（未打开或已关闭） |

### 函数: health()

完整的 socket 健康检测。依次检查 SO_ERROR、available、peek 三个维度，综合判断 socket 是否可安全复用。

```cpp
[[nodiscard]] socket_state health(const net::ip::tcp::socket &s);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `s` | `const net::ip::tcp::socket &` | 待检测的 TCP socket |

**返回值**: `socket_state` — 健康状态

**检测流程**：
1. `s.is_open()` → 无效则返回 `invalid`
2. `getsockopt(SO_ERROR)` → 有错误则返回 `error`
3. `s.available()` → 有错误则返回 `error`
4. `available == 0` → 非阻塞 `recv(MSG_PEEK)`：
   - `would_block`/`try_again` → 返回 `healthy`
   - `n == 0` → 返回 `fin`（对端已发 FIN）
   - 其他 → 返回 `has_data`
5. `available > 0` → 返回 `has_data`

### 函数: healthy_fast()

快速 socket 健康检测。检查 SO_ERROR、available 和 FIN 状态。当 `available == 0` 时通过非阻塞 `recv(MSG_PEEK)` 检测对端是否已发 FIN。用于连接池的缓存命中和归还场景。

```cpp
[[nodiscard]] bool healthy_fast(const net::ip::tcp::socket &s);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `s` | `const net::ip::tcp::socket &` | 待检测的 TCP socket |

**返回值**: `bool` — `true` 表示健康，`false` 表示不健康

**检测流程**：
1. `s.is_open()` → 无效则返回 `false`
2. `getsockopt(SO_ERROR)` → 有错误则返回 `false`
3. `s.available()` → 有错误则返回 `false`
4. `available > 0` → 返回 `false`（有残留数据，不适合复用）
5. `available == 0` → 非阻塞 `recv(MSG_PEEK)`：
   - `would_block`/`try_again` → 返回 `true`（无数据无 FIN，健康）
   - `n == 0` → 返回 `false`（对端已发 FIN）
   - 其他 → 返回 `false`

## 调用链

### 被调用（向上）

| 调用者 | 调用点 | 说明 |
|--------|--------|------|
| [[core/connect/pool/pool|pool]] | `async_acquire()` | 缓存命中时健康检测 |
| [[core/connect/pool/pool|pool]] | `recycle()` | 归还时健康检测 |

### 调用（向下）

| 调用目标 | 调用方式 | 说明 |
|----------|----------|------|
| `getsockopt()` | `SOL_SOCKET, SO_ERROR` | 检查待处理错误 |
| `socket.available()` | Boost.Asio | 检查可用数据量 |
| `socket.receive()` | `MSG_PEEK` | 非阻塞 peek 检测 FIN |

## 使用场景

| 场景 | 调用函数 | 说明 |
|------|----------|------|
| 连接池缓存命中 | `healthy_fast()` | `async_acquire()` 从栈中弹出连接时检测 |
| 连接池归还 | `healthy_fast()` | `recycle()` 归还前检测 |
| 详细诊断 | `health()` | 需要区分具体不健康原因时 |

## 跨平台实现

- **Windows**: 使用 `winsock2.h`，`getsockopt` 的 `len` 参数为 `int`
- **Linux/macOS**: 使用 `sys/socket.h`，`getsockopt` 的 `len` 参数为 `socklen_t`
- 两者通过 `#ifdef _WIN32` 条件编译处理

## 参见

- [[core/connect/overview|connect overview]] — 通道阶段概述
- [[core/connect/pool/pool|pool]] — TCP 连接池
- [[core/transport/reliable|reliable]] — TCP 可靠传输