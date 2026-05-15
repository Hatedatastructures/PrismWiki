---
title: "pool — TCP 连接池"
source: "include/prism/channel/connection/pool.hpp"
module: "channel"
type: api
tags: [channel, connection, pool, 连接池, tcp, 复用]
created: 2026-05-15
updated: 2026-05-15
related:
  - channel/health
  - channel/transport/reliable
  - channel/eyeball/racer
  - memory/container
  - ref/network/tcp
  - ref/network/connection-pool
  - ref/programming/boost-asio
  - ref/memory/pmr
---

# pool.hpp

> 源码: `include/prism/channel/connection/pool.hpp`
> 实现: `src/prism/channel/connection/pool.cpp`
> 模块: [[channel|channel]]

## 概述

[[ref/network/tcp|TCP]] 连接池。管理复用 TCP 连接，减少握手开销。核心机制包括 LIFO 栈式缓存、[[channel/health|僵尸检测]]、线程隔离和后台定时清理。

设计约束：
- 连接池设计为线程局部使用，不支持跨线程共享
- 必须确保 `io_context` 在生命周期内保持运行

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|memory]] | [[ref/memory/pmr|PMR]] 容器 |
| 依赖 | [[channel/health|health]] | `healthy_fast()` 健康检测 |
| 被依赖 | [[channel/transport/reliable|reliable]] | 连接池复用 |
| 被依赖 | [[channel/eyeball/racer|racer]] | 竞速器使用连接池 |

## 命名空间

`psm::channel`

## 结构体: endpoint_key

端点键，用于唯一标识一个 TCP 端点（IP + 端口），是连接池缓存的 Key。

| 字段 | 类型 | 说明 |
|------|------|------|
| `port` | `uint16_t` | 端口号 |
| `family` | `uint8_t` | 协议族：4=IPv4, 6=IPv6 |
| `address` | `array<uchar, 16>` | IP 地址，IPv4 使用前 4 字节 |

支持 `operator==` 默认比较。

---

## 自由函数

### make_endpoint_key()

**功能说明**：将 Boost.Asio 的 `tcp::endpoint` 转换为内部的 `endpoint_key` 结构，提取地址、端口和协议族信息。

**签名**：
```cpp
endpoint_key make_endpoint_key(const tcp::endpoint &endpoint) noexcept;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `endpoint` | `const tcp::endpoint &` | TCP 端点 |

**返回值**：`endpoint_key` — 端点键

**调用（向下）**：无

**被调用（向上）**：
- `async_acquire()` 缓存查找
- `recycle()` 缓存归还

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

## 结构体: endpoint_hash

端点键哈希函数，使用 FNV-1a 变体算法。

### operator()()

**功能说明**：使用 FNV-1a 变体算法计算端点键的哈希值，一次性处理 port、family 和 address 所有字段。

**签名**：
```cpp
std::size_t operator()(const endpoint_key &key) const noexcept;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `key` | `const endpoint_key &` | 端点键 |

**返回值**：`std::size_t` — 哈希值

**调用（向下）**：无

**被调用（向上）**：[[ref/memory/pmr|PMR]] `unordered_map` 内部使用

**涉及的知识域**：[[ref/memory/pmr|PMR]]

---

## 结构体: config

连接池配置，控制连接池的行为参数。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_cache_per_endpoint` | `uint32_t` | `32` | 单端点最大缓存连接数 |
| `connect_timeout_ms` | `uint64_t` | `300` | 连接超时（毫秒） |
| `max_idle_seconds` | `uint64_t` | `30` | 空闲连接最大存活时间（秒） |
| `cleanup_interval_sec` | `uint64_t` | `10` | 后台清理间隔（秒） |
| `recv_buffer_size` | `uint32_t` | `65536` | 接收缓冲区大小 |
| `send_buffer_size` | `uint32_t` | `65536` | 发送缓冲区大小 |
| `tcp_nodelay` | `bool` | `true` | 启用 TCP_NODELAY |
| `keep_alive` | `bool` | `true` | 启用 SO_KEEPALIVE |
| `cache_ipv6` | `bool` | `false` | 缓存 IPv6 连接 |

---

## 类: pooled_connection

连接池连接的 RAII 包装器。内联存储 pool 指针、socket 指针和 endpoint，零堆分配。析构时自动归还连接到连接池。

### 构造函数

---

#### pooled_connection()

**功能说明**：默认构造一个无效的空连接包装器，所有指针为空。

**签名**：
```cpp
pooled_connection() = default;
```

**参数**：无

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`async_acquire()` 返回失败时构造空连接

**涉及的知识域**：无

---

#### pooled_connection(pool, socket, endpoint)

**功能说明**：使用连接池指针、socket 指针和目标端点构造连接包装器。

**签名**：
```cpp
pooled_connection(connection_pool *pool, tcp::socket *socket, tcp::endpoint endpoint);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `pool` | `connection_pool *` | 关联的连接池指针 |
| `socket` | `tcp::socket *` | 持有的 socket 指针 |
| `endpoint` | `tcp::endpoint` | 关联的目标端点 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`async_acquire()` 命中缓存或新建连接后构造

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

#### ~pooled_connection()

**功能说明**：析构时自动调用 `reset()` 归还或关闭连接。

**签名**：
```cpp
~pooled_connection();
```

**参数**：无

**返回值**：无

**调用（向下）**：`reset()`

**被调用（向上）**：RAII 作用域结束

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

#### pooled_connection(pooled_connection &&)

**功能说明**：移动构造函数，转移连接的所有权，移动后源对象变为无效状态。

**签名**：
```cpp
pooled_connection(pooled_connection &&other) noexcept;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `other` | `pooled_connection &&` | 要移动的连接包装器 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：`co_return` 返回连接、[[channel/transport/reliable|reliable]] 构造

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

#### operator=(pooled_connection &&)

**功能说明**：移动赋值运算符，转移所有权。先 reset 当前持有的连接，再接管 other 的资源。

**签名**：
```cpp
pooled_connection &operator=(pooled_connection &&other) noexcept;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `other` | `pooled_connection &&` | 要移动的连接包装器 |

**返回值**：`pooled_connection &` — 当前对象的引用

**调用（向下）**：`reset()`

**被调用（向上）**：连接赋值场景

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

### 成员函数

---

#### get()

**功能说明**：返回持有的 socket 指针，无效时返回 `nullptr`。

**签名**：
```cpp
[[nodiscard]] tcp::socket *get() const noexcept;
```

**参数**：无

**返回值**：`tcp::socket *` — socket 指针

**调用（向下）**：无

**被调用（向上）**：[[channel/transport/reliable|reliable]] `close()` 中取消异步操作

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### operator*()

**功能说明**：解引用 socket，调用前必须确保 `valid()` 为 `true`。

**签名**：
```cpp
[[nodiscard]] tcp::socket &operator*() const noexcept;
```

**参数**：无

**返回值**：`tcp::socket &` — socket 引用

**调用（向下）**：无

**被调用（向上）**：[[channel/transport/reliable|reliable]] `native_socket()` 中访问池中 socket

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### operator->()

**功能说明**：返回 socket 指针，用于通过箭头操作符访问成员。

**签名**：
```cpp
[[nodiscard]] tcp::socket *operator->() const noexcept;
```

**参数**：无

**返回值**：`tcp::socket *` — socket 指针

**调用（向下）**：无

**被调用（向上）**：通过箭头操作符调用 socket 方法

**涉及的知识域**：[[ref/network/tcp|TCP]]

---

#### valid()

**功能说明**：检查 socket 指针是否非空。

**签名**：
```cpp
[[nodiscard]] bool valid() const noexcept;
```

**参数**：无

**返回值**：`bool` — 有效返回 `true`

**调用（向下）**：无

**被调用（向上）**：[[channel/transport/reliable|reliable]] `close()` 中判断是否为池连接

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

#### operator bool()

**功能说明**：隐式转换为 `bool`，等同于 `valid()`。

**签名**：
```cpp
[[nodiscard]] explicit operator bool() const noexcept;
```

**参数**：无

**返回值**：`bool` — 有效返回 `true`

**调用（向下）**：`valid()`

**被调用（向上）**：条件判断场景

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

#### release()

**功能说明**：释放连接所有权，不归还到连接池。调用方接管 socket 指针并负责关闭。调用后 `valid()` 返回 `false`。

**签名**：
```cpp
[[nodiscard]] tcp::socket *release() noexcept;
```

**参数**：无

**返回值**：`tcp::socket *` — socket 指针，调用方负责关闭

**调用（向下）**：无

**被调用（向上）**：需要接管连接所有权的场景

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

#### reset()

**功能说明**：归还或关闭连接。如果有关联的连接池，调用 `recycle()` 进行健康检测和可能的复用；否则直接关闭并释放 socket。

**签名**：
```cpp
void reset();
```

**参数**：无

**返回值**：无

**调用（向下）**：`pool_->recycle(socket_, endpoint_)` 或 `socket_->close()` + `delete socket_`

**被调用（向上）**：析构函数、移动赋值

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

## 结构体: pool_stats

连接池统计信息，用于监控和诊断。

| 字段 | 类型 | 说明 |
|------|------|------|
| `idle_count` | `size_t` | 当前空闲连接数 |
| `endpoint_count` | `size_t` | 有缓存的端点数 |
| `total_acquires` | `size_t` | 总获取次数 |
| `total_hits` | `size_t` | 缓存命中次数 |
| `total_creates` | `size_t` | 新建连接次数 |
| `total_recycles` | `size_t` | 归还次数 |
| `total_evictions` | `size_t` | 驱逐次数（容量满/不健康/过期） |

---

## 类: connection_pool

> 源码: `include/prism/channel/connection/pool.hpp:239`

### 构造函数

---

#### connection_pool(ioc, resource, config)

**功能说明**：使用 IO 上下文和可选的内存资源、配置参数初始化连接池。

**签名**：
```cpp
explicit connection_pool(net::io_context &ioc,
                         const memory::resource_pointer resource = memory::current_resource(),
                         const config &config = {});
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `ioc` | `net::io_context &` | IO 上下文，用于异步操作和定时器 |
| `resource` | `const memory::resource_pointer` | [[ref/memory/pmr|PMR]] 内存资源 |
| `config` | `const config &` | 连接池配置 |

**返回值**：无

**调用（向下）**：无

**被调用（向上）**：worker 启动时创建连接池

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]], [[ref/memory/pmr|PMR]]

---

#### ~connection_pool()

**功能说明**：析构函数，调用 `clear()` 关闭并释放所有缓存连接。

**签名**：
```cpp
~connection_pool();
```

**参数**：无

**返回值**：无

**调用（向下）**：`clear()`

**被调用（向上）**：worker 销毁时

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

### 成员函数

---

#### async_acquire()

**功能说明**：获取一个 TCP 连接。优先从 LIFO 栈缓存中复用，依次进行过期检查和健康检测；缓存未命中时通过 `co_spawn + timer` 创建新连接，超时由 `config::connect_timeout_ms` 控制。新建连接自动设置 TCP_NODELAY、SO_KEEPALIVE 等选项。

**签名**：
```cpp
[[nodiscard]] auto async_acquire(tcp::endpoint endpoint)
    -> net::awaitable<std::pair<fault::code, pooled_connection>>;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `endpoint` | `tcp::endpoint` | 目标 TCP 端点 |

**返回值**：`net::awaitable<std::pair<fault::code, pooled_connection>>` — 错误码和连接包装器，成功时 `code` 为 `success` 且连接有效

**调用（向下）**：
- `make_endpoint_key()` — 构造缓存键
- `healthy_fast()` — 健康检测
- `tcp::socket::async_connect()` — 建立新连接
- `socket->set_option()` — 设置 socket 选项

**被调用（向上）**：
- [[channel/eyeball/racer|racer]] `race()` 和 `race_endpoint()` 中获取连接
- 路由器建立上游连接

**涉及的知识域**：[[ref/network/connection-pool|连接池]], [[ref/network/tcp|TCP]]

**async_acquire 流程**：
1. 从缓存中查找端点对应的 LIFO 栈
2. LIFO 弹出栈顶连接 → 过期检查（`max_idle_seconds`） → `healthy_fast()` 健康检测 → 返回
3. 栈空则移除端点条目
4. 缓存未命中 → `new tcp::socket` + `async_connect` + `steady_timer` 竞速
5. 超时返回 `fault::code::timeout`，连接失败返回 `fault::code::bad_gateway`
6. 成功后设置 `TCP_NODELAY`、`SO_KEEPALIVE`、缓冲区大小

---

#### recycle()

**功能说明**：归还连接到连接池（内部接口，由 `pooled_connection::reset()` 调用）。执行 IPv6 过滤、健康检测和容量检查，满足条件的连接入栈等待复用，不满足的直接关闭。

**签名**：
```cpp
void recycle(tcp::socket *s, const tcp::endpoint &endpoint);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `s` | `tcp::socket *` | 待归还的 socket 指针 |
| `endpoint` | `const tcp::endpoint &` | 关联的目标端点 |

**返回值**：无

**调用（向下）**：
- `make_endpoint_key()` — 构造缓存键
- `healthy_fast()` — 健康检测
- `delete_socket()` — 关闭不健康连接

**被调用（向上）**：`pooled_connection::reset()` — 析构或显式归还时

**涉及的知识域**：[[ref/network/connection-pool|连接池]], [[channel/health|health]]

**recycle 流程**：
1. 无效连接直接销毁
2. IPv6 连接默认不缓存（受 `config_.cache_ipv6` 控制）
3. `healthy_fast()` 检测不通过则销毁
4. 缓存容量检查（`max_cache_per_endpoint`），超限则销毁
5. 入栈并记录 `last_used` 时间

---

#### start()

**功能说明**：启动后台清理协程，按 `config::cleanup_interval_sec` 指定的间隔周期性调用 `cleanup()` 移除过期连接。重复调用无效。

**签名**：
```cpp
void start();
```

**参数**：无

**返回值**：无

**调用（向下）**：
- `cleanup()` — 定期移除过期连接
- `co_spawn()` — 投递清理协程

**被调用（向上）**：worker 初始化时调用

**涉及的知识域**：[[ref/programming/boost-asio|Boost.Asio]]

---

#### stats()

**功能说明**：返回统计信息快照。`idle_count` 和 `endpoint_count` 为实时计算值，其余为累计计数器。

**签名**：
```cpp
[[nodiscard]] auto stats() const -> pool_stats;
```

**参数**：无

**返回值**：`pool_stats` — 统计信息快照

**调用（向下）**：无

**被调用（向上）**：监控和诊断

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

#### get_config()

**功能说明**：返回当前连接池的配置参数。

**签名**：
```cpp
[[nodiscard]] const config &get_config() const noexcept;
```

**参数**：无

**返回值**：`const config &` — 配置常量引用

**调用（向下）**：无

**被调用（向上）**：查询配置

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

### 私有方法

---

#### clear()

**功能说明**：清理所有缓存连接。取消后台清理定时器，设置退出标志，关闭并释放所有缓存中的 socket，清空缓存容器。

**签名**：
```cpp
void clear();
```

**参数**：无

**返回值**：无

**调用（向下）**：`delete_socket()` — 关闭每个 socket

**被调用（向上）**：析构函数

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

---

#### cleanup()

**功能说明**：后台清理，移除过期连接。遍历所有端点的连接栈，使用原地压缩算法移除超过 `max_idle_seconds` 的过期连接，空栈的端点条目一并移除。

**签名**：
```cpp
void cleanup();
```

**参数**：无

**返回值**：无

**调用（向下）**：`delete_socket()` — 关闭过期 socket

**被调用（向上）**：`start()` 启动的后台协程定期调用

**涉及的知识域**：[[ref/network/connection-pool|连接池]]

### 调用链

```
router::async_forward → connection_pool::async_acquire → TCP connect
pooled_connection 析构 → connection_pool::recycle → 健康检测 → 入栈或关闭
connection_pool::start → cleanup 协程 → 定期移除过期连接
```
