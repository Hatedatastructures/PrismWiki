---
title: "health — Socket 健康检测"
source: "include/prism/channel/health.hpp"
module: "channel"
type: api
tags: [channel, health, socket, 连接池, 健康检测]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/connection/pool
  - channel/transport/reliable
  - ref/network/tcp
  - ref/network/connection-pool
  - ref/programming/boost-asio
---

# health.hpp

> 源码: `include/prism/channel/health.hpp`
> 实现: `src/prism/channel/health.cpp`
> 模块: [[channel|channel]]

## 概述

Socket 健康检测模块。提供 [[ref/network/tcp|TCP]] socket 健康状态检测功能，用于 [[channel/connection/pool|连接池]] 复用前的有效性验证。

检测维度：
1. **SO_ERROR** — 获取 socket 的待处理错误码
2. **available** — 检查是否有待读取数据
3. **peek** — 通过 `MSG_PEEK` 预读检测对端是否已发 FIN

健康状态分类：
- `healthy` — 连接健康可安全复用
- `has_data` — 有待读数据，可能是延迟数据或 FIN
- `fin` — 对端已关闭，不可复用
- `error` — socket 发生错误，不可复用
- `invalid` — socket 无效（未打开或已关闭）

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[channel/connection/pool|pool]] | `async_acquire()` 和 `recycle()` 调用健康检测 |
| 被依赖 | [[channel/transport/reliable|reliable]] | 连接池复用时间接使用 |

## 命名空间

`psm::channel`

## 枚举: socket_state

Socket 健康状态枚举。

```cpp
enum class socket_state { healthy, has_data, fin, error, invalid };
```

| 值 | 说明 |
|----|------|
| `healthy` | 健康，可安全复用 |
| `has_data` | 有待读数据（可能是延迟数据或 FIN） |
| `fin` | 对端已关闭，不可复用 |
| `error` | socket 发生错误，不可复用 |
| `invalid` | socket 无效（未打开或已关闭） |

---

## 函数

### health()

**功能说明**：完整的 socket 健康检测，依次检查 SO_ERROR、available、peek 三个维度，综合判断 socket 是否可安全复用。

**签名**：
```cpp
[[nodiscard]] socket_state health(const net::ip::tcp::socket &s);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `s` | `const net::ip::tcp::socket &` | 待检测的 TCP socket |

**返回值**：`socket_state` — 健康状态

**调用（向下）**：
- `::getsockopt(SOL_SOCKET, SO_ERROR)` — 检查待处理错误
- `socket.available()` — 检查可用数据量
- `socket.receive(buffer, message_peek)` — 非阻塞 peek 检测 FIN

**被调用（向上）**：需要详细健康状态的诊断场景

**涉及的知识域**：[[ref/network/tcp|TCP]]

**检测流程**：
1. `s.is_open()` → 无效则返回 `invalid`
2. `getsockopt(SO_ERROR)` → 有错误则返回 `error`
3. `s.available()` → 有错误则返回 `error`
4. `available == 0` → 非阻塞 `recv(MSG_PEEK)`：
   - `would_block`/`try_again` → 返回 `healthy`
   - `n == 0` → 返回 `fin`（对端已发 FIN）
   - 其他 → 返回 `has_data`
5. `available > 0` → 返回 `has_data`

---

### healthy_fast()

**功能说明**：快速 socket 健康检测，检查 SO_ERROR、available 和 FIN 状态。当 `available == 0` 时通过非阻塞 `recv(MSG_PEEK)` 检测对端是否已发 FIN。用于 [[channel/connection/pool|连接池]] 的缓存命中和归还场景。

**签名**：
```cpp
[[nodiscard]] bool healthy_fast(const net::ip::tcp::socket &s);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `s` | `const net::ip::tcp::socket &` | 待检测的 TCP socket |

**返回值**：`bool` — `true` 表示健康，`false` 表示不健康

**调用（向下）**：
- `::getsockopt(SOL_SOCKET, SO_ERROR)` — 检查待处理错误
- `socket.available()` — 检查可用数据量
- `socket.receive(buffer, message_peek)` — 非阻塞 peek 检测 FIN

**被调用（向上）**：
- [[channel/connection/pool|pool]] `async_acquire()` 缓存命中时健康检测
- [[channel/connection/pool|pool]] `recycle()` 归还时健康检测

**涉及的知识域**：[[ref/network/tcp|TCP]], [[ref/network/connection-pool|连接池]]

**检测流程**：
1. `s.is_open()` → 无效则返回 `false`
2. `getsockopt(SO_ERROR)` → 有错误则返回 `false`
3. `s.available()` → 有错误则返回 `false`
4. `available > 0` → 返回 `false`（有残留数据，不适合复用）
5. `available == 0` → 非阻塞 `recv(MSG_PEEK)`：
   - `would_block`/`try_again` → 返回 `true`（无数据无 FIN，健康）
   - `n == 0` → 返回 `false`（对端已发 FIN）
   - 其他 → 返回 `false`

## 使用场景

| 场景 | 调用函数 | 说明 |
|------|----------|------|
| 连接池缓存命中 | `healthy_fast()` | `async_acquire()` 从栈中弹出连接时检测 |
| 连接池归还 | `healthy_fast()` | `recycle()` 归还前检测 |
| 详细诊断 | `health()` | 需要区分具体不健康原因时 |

## 跨平台实现

- **Windows**：使用 `winsock2.h`，`getsockopt` 的 `len` 参数为 `int`
- **Linux/macOS**：使用 `sys/socket.h`，`getsockopt` 的 `len` 参数为 `socklen_t`
- 两者都通过 `#ifdef _WIN32` 条件编译处理
