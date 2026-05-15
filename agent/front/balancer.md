---
title: "balancer.hpp — 前端代理负载均衡器"
source: "include/prism/agent/front/balancer.hpp"
module: "agent"
type: api
tags: [agent, front, balancer, 负载均衡]
related: [agent/front/listener, agent/worker/worker, agent/worker/stats]
created: 2026-05-15
updated: 2026-05-15
---

# balancer.hpp

> 源码: `include/prism/agent/front/balancer.hpp` / `src/prism/agent/front/balancer.cpp`
> 模块: [[agent|Agent]] / front

## 概述

实现基于加权评分的工作线程选择算法，支持过载检测与全局反压机制。负载均衡器通过收集各工作线程的实时负载快照，计算综合评分后选择最优目标进行连接分发。评分公式综合考虑活跃会话数、待处理移交数和事件循环延迟三个维度，权重可配置。当所有工作线程均进入过载状态时，系统将触发全局反压，暂停接受新连接。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 被依赖 | [[agent/front/listener|listener]] | 监听器调用 `select()` 与 `dispatch()` 分发连接 |
| 被依赖 | [[agent/worker/worker|worker]] | worker 注册绑定信息到 balancer |

## 命名空间

`psm::agent::front`

---

## 结构体: worker_load_snapshot

### 概述
工作线程负载快照，记录某一时刻工作线程的负载状态。快照数据由工作线程定期上报，通过 `worker_binding::snapshot` 回调获取。

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `std::uint32_t` | `active_sessions` | 当前活跃的会话数量 |
| `std::uint32_t` | `pending_handoffs` | 等待处理的移交任务数 |
| `std::uint64_t` | `event_loop_lag_us` | 事件循环延迟，单位微秒 |

---

## 结构体: distribute_config

### 概述
分发策略配置参数。过载检测采用滞后机制避免抖动，进入过载阈值高于退出阈值。

### 成员变量
| 类型 | 名称 | 默认值 | 说明 |
|------|------|--------|------|
| `double` | `enter_overload` | 0.90 | 进入过载状态的负载阈值 |
| `double` | `exit_overload` | 0.80 | 退出过载状态的负载阈值 |
| `double` | `global_backpressure_threshold` | 0.95 | 全局反压触发阈值 |
| `double` | `weight_session` | 0.60 | 会话数权重 |
| `double` | `weight_pending` | 0.10 | 待处理数权重 |
| `double` | `weight_lag` | 0.30 | 延迟权重 |
| `std::uint32_t` | `session_capacity` | 1024 | 会话容量基准值 |
| `std::uint32_t` | `pending_capacity` | 256 | 待处理容量基准值 |
| `std::uint64_t` | `lag_capacity_us` | 5000 | 延迟容量基准值（微秒） |

---

## 类: balancer

### 概述
工作线程负载均衡器，维护所有工作线程的绑定信息与过载状态。选择算法首先根据亲和性值计算候选工作线程，然后获取其实时负载快照并计算评分，选择评分最低的健康工作线程进行分发。

### 内部结构体

#### `worker_binding`
工作线程绑定信息，封装分发函数与负载快照获取函数。

| 类型 | 名称 | 说明 |
|------|------|------|
| `std::function<void(tcp::socket)>` | `dispatch` | 连接分发函数，将 socket 移交至目标 worker 的事件循环 |
| `std::function<worker_load_snapshot()>` | `snapshot` | 负载快照获取函数，返回 worker 当前负载状态 |

#### `select_result`
工作线程选择结果。

| 类型 | 名称 | 说明 |
|------|------|------|
| `std::size_t` | `worker_index` | 选中的工作线程索引 |
| `bool` | `overflowed` | 选中的 primary worker 是否已过载 |
| `bool` | `backpressure` | 是否触发全局反压（所有 worker 过载或最低评分超过阈值） |

### 成员变量
| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::vector<worker_binding>` | `bindings_` | 工作线程绑定列表 |
| `memory::vector<std::uint8_t>` | `overload_state_` | 过载状态标记（0=正常，1=过载） |
| `distribute_config` | `config_` | 分发配置参数 |
| `memory::resource_pointer` | `mr_` | 内存资源指针 |

---

## 函数文档

### balancer() 构造函数

#### 功能说明
初始化负载均衡器实例。将传入的绑定信息列表和配置参数移动/拷贝至成员变量，使用指定的 PMR 内存资源分配内部容器，并为每个 worker 初始化过载状态标记为正常（0）。绑定信息列表不能为空，否则后续选择操作将产生未定义行为。

#### 签名
```cpp
explicit balancer(memory::vector<worker_binding> bindings, const distribute_config &config = {},
                  memory::resource_pointer mr = memory::current_resource());
```

#### 参数

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `bindings` | `memory::vector<worker_binding>` | in | 工作线程绑定信息列表，包含每个 worker 的分发函数和快照获取函数 |
| `config` | `const distribute_config &` | in | 分发策略配置参数，使用默认值时各项权重和阈值取预设值 |
| `mr` | `memory::resource_pointer` | in | 内存资源指针，用于分配 `bindings_` 和 `overload_state_` 容器 |

#### 返回值
无（构造函数）。

#### 调用（向下）
- 无函数调用。仅执行成员初始化和 `overload_state_.resize()`。

#### 被调用（向上）
- [[agent/front/listener|`listener` 构造函数]] -- 在启动阶段创建 balancer 实例，传入从 worker 收集的绑定信息。

#### 知识域
[[memory/container|PMR 内存策略]] -- 内部容器使用 PMR 分配器，`mr` 参数决定内存来源。

---

### select()

#### 功能说明
核心选择算法：为新连接选择最优工作线程。算法分为四步：(1) 使用 splitmix64 哈希函数对亲和性值进行两次混合，分别选出 primary 和 secondary 候选 worker；(2) 遍历所有 worker 采集负载快照、计算评分并更新过载状态；(3) 若 primary 未过载则保持亲和性选择 primary，否则选择 primary 和 secondary 中评分更低者；(4) 判断全局反压条件（所有 worker 过载或最低评分超过全局阈值）。该函数标记为 `noexcept`，可安全在热路径中调用。

#### 签名
```cpp
[[nodiscard]] auto select(std::uint64_t affinity_value) noexcept -> select_result;
```

#### 参数

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `affinity_value` | `std::uint64_t` | in | 亲和性哈希值（来自客户端 IP 地址哈希），用于一致性地将同一客户端分配到同一 worker |

#### 返回值

| 类型 | 说明 |
|------|------|
| `select_result` | 包含选中的 worker 索引、primary 是否过载、是否触发全局反压 |

#### 调用（向下）
- [[agent/front/balancer#mix_hash|`mix_hash()`]] -- 对亲和性值进行两次 splitmix64 混合，生成 primary 和 secondary 候选索引。
- [[agent/front/balancer#score|`score()`]] -- 对每个 worker 的负载快照计算加权评分。
- [[agent/front/balancer#refresh_state|`refresh_state()`]] -- 根据评分和滞后阈值更新每个 worker 的过载状态。
- `worker_binding::snapshot()` -- 获取每个 worker 的实时负载快照。

#### 被调用（向上）
- [[agent/front/listener|`listener::accept_loop()`]] -- 在接受新连接后调用 `select()` 获取目标 worker 索引，再调用 `dispatch()` 完成分发。

#### 知识域
负载均衡算法 -- splitmix64 哈希确保客户端均匀分布；迟滞机制防止过载状态抖动。

---

### dispatch()

#### 功能说明
将已接受的 TCP 套接字实际移交给选中的工作线程。通过调用目标 worker 的绑定分发函数，将 socket 所有权转移至其事件循环。分发后调用方不应再访问该套接字。索引越界时回退到 worker 0 作为安全兜底，空绑定列表时静默返回。

#### 签名
```cpp
void dispatch(std::size_t worker_index, tcp::socket socket) const;
```

#### 参数

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `worker_index` | `std::size_t` | in | 目标工作线程索引，由 `select()` 返回 |
| `socket` | `tcp::socket` | in | 待分发的 TCP 套接字，所有权通过 move 转移 |

#### 返回值
无。

#### 调用（向下）
- `worker_binding::dispatch()` -- 调用目标 worker 注册的分发回调，将 socket 移交至其 `io_context`。

#### 被调用（向上）
- [[agent/front/listener|`listener::accept_loop()`]] -- 在 `select()` 之后调用 `dispatch()`，将 socket 移交至选中的 worker。

#### 知识域
[[agent/worker/worker|Worker 事件循环]] -- socket 被投递到 worker 的 `io_context`，由 [[agent/worker/launch|launch]] 启动会话。

---

### size()

#### 功能说明
返回当前注册的工作线程数量。该值在 balancer 构造后固定不变，等于 `bindings_` 向量的大小。

#### 签名
```cpp
[[nodiscard]] auto size() const noexcept -> std::size_t;
```

#### 参数
无。

#### 返回值

| 类型 | 说明 |
|------|------|
| `std::size_t` | 工作线程数量 |

#### 调用（向下）
- 无。

#### 被调用（向上）
- 外部模块可用于查询 worker 总数，例如统计和诊断场景。

#### 知识域
[[agent/worker/worker|Worker 生命周期]] -- balancer 的 worker 数量在构造时由 `listener` 确定，等于 CPU 核心数减一。

---

### mix_hash() (私有静态)

#### 功能说明
splitmix64 哈希混合函数，将输入值通过三轮移位-乘法-异或操作进行混淆，提高哈希分布的随机性。该算法源自 MurmurHash3 的 finalize 混合步骤。`select()` 使用不同种子调用此函数两次，生成两个不同的候选 worker 索引，避免客户端 IP 集中在某子网时造成分配聚集。

#### 签名
```cpp
[[nodiscard]] static constexpr auto mix_hash(std::uint64_t value) noexcept -> std::uint64_t;
```

#### 参数

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `value` | `std::uint64_t` | in | 输入哈希值（客户端亲和性值或其异或变体） |

#### 返回值

| 类型 | 说明 |
|------|------|
| `std::uint64_t` | 混合后的哈希值，分布更加均匀 |

#### 调用（向下）
- 无。纯算术运算，无外部依赖。

#### 被调用（向上）
- [[agent/front/balancer#select|`select()`]] -- 被调用两次：一次生成 primary 候选索引，一次（使用异或种子）生成 secondary 候选索引。

#### 知识域
负载均衡算法 -- splitmix64 是确定性哈希，相同输入始终产生相同输出，保证同一客户端的亲和性一致性。

---

### score() (私有)

#### 功能说明
计算单个 worker 的综合负载评分。将负载快照中的三个维度（活跃会话数、待处理移交数、事件循环延迟）分别除以各自的容量基准值归一化为比率，再按配置权重加权求和。评分值在 0.0（空闲）到约 1.0（满载）之间，值越低表示负载越轻。使用比率而非绝对值使得不同容量配置可以公平比较。

#### 签名
```cpp
[[nodiscard]] auto score(const worker_load_snapshot &snapshot) const noexcept -> double;
```

#### 参数

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `snapshot` | `const worker_load_snapshot &` | in | 工作线程负载快照，包含活跃会话数、待处理数和事件循环延迟 |

#### 返回值

| 类型 | 说明 |
|------|------|
| `double` | 综合负载评分，0.0 表示空闲，值越高表示负载越重 |

#### 调用（向下）
- 读取 `config_` 中的权重和容量参数进行加权计算。

#### 被调用（向上）
- [[agent/front/balancer#select|`select()`]] -- 在遍历所有 worker 时对每个 worker 调用 `score()` 获取负载评分。

#### 知识域
[[agent/worker/stats|Worker 统计指标]] -- 评分所用的三个维度（活跃会话数、待处理移交数、事件循环延迟）由 [[agent/worker/stats|worker stats]] 模块采集上报。

---

### refresh_state() (私有)

#### 功能说明
根据当前负载评分更新指定 worker 的过载状态标记。采用迟滞（hysteresis）机制避免状态抖动：当 worker 处于正常状态且评分达到或超过 `enter_overload` 阈值时标记为过载；当 worker 处于过载状态且评分降至 `exit_overload` 阈值以下时恢复为正常。两个阈值之间的差值形成"死区"，防止负载在阈值附近波动时频繁切换。

#### 签名
```cpp
void refresh_state(std::size_t worker_index, double load_score) noexcept;
```

#### 参数

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `worker_index` | `std::size_t` | in | 工作线程索引 |
| `load_score` | `double` | in | 当前负载评分，由 `score()` 计算 |

#### 返回值
无。

#### 调用（向下）
- 读写 `overload_state_` 数组更新过载标记。

#### 被调用（向上）
- [[agent/front/balancer#select|`select()`]] -- 在遍历所有 worker 时对每个 worker 调用 `refresh_state()` 更新过载状态。

#### 知识域
负载均衡算法 -- 迟滞机制是控制论中的经典去抖动策略，`enter_overload`（0.90）与 `exit_overload`（0.80）之间的差值防止状态频繁翻转。

---

## 调用链

```
[[agent/front/listener|listener::accept_loop]]
  │
  ├─ balancer::select(affinity)
  │    ├─ mix_hash(affinity)              → primary 候选索引
  │    ├─ mix_hash(affinity ^ seed)       → secondary 候选索引
  │    ├─ [遍历所有 worker]
  │    │    ├─ worker_binding::snapshot()  → 获取负载快照
  │    │    ├─ score(snapshot)             → 计算加权评分
  │    │    └─ refresh_state(index, score) → 更新过载状态
  │    └─ 返回 select_result
  │
  └─ balancer::dispatch(index, socket)
       └─ worker_binding::dispatch(socket) → 移交至 worker 事件循环
            └─ [[agent/worker/launch|launch()]] → 启动会话
```

## 知识域

- 负载均衡算法 -- 加权评分选择 + splitmix64 哈希亲和性 + 迟滞过载检测
- [[agent/worker/stats|Worker 统计]] -- 负载快照的三个维度由 worker stats 模块提供
- [[ref/network/tcp|反压机制]] -- 全局反压阈值触发时暂停接受新连接
