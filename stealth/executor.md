---
title: "executor — 伪装方案执行器"
source: "include/prism/stealth/executor.hpp"
module: "stealth"
type: api
tags: [stealth, executor, 执行器, 方案管道]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/registry|registry]]"
  - "[[stealth/scheme|scheme]]"
  - "[[stealth/native|native]]"
  - "[[recognition/recognition|recognition]]"
  - "[[stealth/reality/scheme|reality/scheme]]"
---

# executor.hpp

> 源码: `include/prism/stealth/executor.hpp`
> 实现: `src/prism/stealth/executor.cpp`
> 模块: [[stealth|stealth]]

## 概述

伪装方案执行器。根据分析结果依次尝试伪装方案，直到某个方案成功。每个方案执行后通过 `detected` 类型判断是否"是我"：返回 TLS 表示不匹配，继续下一个；返回具体协议表示匹配，终止执行。全部失败时返回错误。执行器从 [[stealth/registry|scheme_registry]] 构建，不硬编码方案列表。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[stealth/scheme|scheme]] | 使用 stealth_scheme 基类和 handshake_context/handshake_result |
| 依赖 | [[stealth/registry|registry]] | 使用 scheme_registry 获取方案列表 |
| 依赖 | [[recognition/result|result]] | 使用 analysis_result 候选列表 |
| 依赖 | [[pipeline/primitives|primitives]] | 使用 preview 包装预读数据 |
| 依赖 | [[channel/transport/snapshot|snapshot]] | 使用 snapshot 实现 rewind |
| 被依赖 | [[recognition/recognition|recognition]] | 识别流程中调用执行器 |

## 命名空间

`psm::stealth`

---

## 类: scheme_executor

> 源码: `include/prism/stealth/executor.hpp:28`

### 概述

伪装方案执行器。按候选方案列表依次尝试执行，支持分析驱动模式。从 [[stealth/registry|scheme_registry]] 构建，不硬编码方案列表。

### 设计意图

执行器的核心逻辑是：按候选方案列表依次尝试执行，直到某个方案成功。每个方案执行后通过 `detected` 类型判断是否"是我"：
- 返回 TLS 表示不匹配，继续下一个
- 返回具体协议表示匹配，终止执行
- 全部失败时返回错误

执行器从 [[stealth/registry|scheme_registry]] 构建，不硬编码方案列表，支持动态注册新方案。

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| schemes_ | std::vector\<shared_scheme\> | 方案列表，从注册表复制 |

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| scheme_executor() | 从注册表构建执行器 | 下文 |
| execute_by_analysis() | 按分析结果驱动执行 | 下文 |
| execute() | 按候选列表执行 | 下文 |
| find_scheme() | 按名称查找方案 | 下文 |
| execute_single() | 执行单个方案 | 下文 |
| pass_through() | 传递数据到下一方案 | 下文 |
| ensure_snapshot() | 确保传输层快照 | 下文 |
| try_rewind() | 尝试回退到快照 | 下文 |
| execute_pipeline() | 执行方案管道 | 下文 |

### 生命周期

1. **构造**: 由 [[recognition/recognition|recognition]] 模块从 [[stealth/registry|scheme_registry]] 构建
2. **使用**: 被 recognition 调用，执行方案管道
3. **销毁**: 会话结束时自动销毁

### 线程安全

- 所有函数需要在协程中调用
- 不支持并发执行

### 异常安全

- 所有函数不抛异常，错误通过返回值报告

---

## 函数: scheme_executor()

> 源码: `include/prism/stealth/executor.hpp:35`
> 实现: `src/prism/stealth/executor.cpp:15`

### 功能

从注册表构建执行器。从 [[stealth/registry|scheme_registry]] 中获取所有注册的方案，拷贝到内部 `schemes_` 列表。

### 签名

```cpp
explicit scheme_executor(const scheme_registry &registry);
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| registry | const scheme_registry& | 方案注册表引用 |

### 返回值

无（构造函数）

### 调用（向下）

- [[stealth/registry|scheme_registry::all()]] — 获取所有已注册方案

### 被调用（向上）

- [[recognition/recognition|recognize]] — 识别流程中构建执行器

### 知识域

- [[stealth/registry|scheme_registry]] — 方案注册表单例

---

## 函数: execute_by_analysis()

> 源码: `include/prism/stealth/executor.hpp:45`
> 实现: `src/prism/stealth/executor.cpp:128`

### 功能

按分析结果驱动执行方案管道。从 `analysis_result` 中获取候选方案名称列表，依次尝试执行。候选为空时按注册顺序执行；全部失败则执行 [[stealth/native|native]] 兜底。每个方案返回 TLS 表示 "不是我"，transport 和 preread 数据传递给下一个方案。

### 签名

```cpp
[[nodiscard]] auto execute_by_analysis(const recognition::analysis_result &analysis, handshake_context ctx) const
    -> net::awaitable<handshake_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| analysis | const recognition::analysis_result& | ClientHello 分析结果（含候选方案名称列表） |
| ctx | handshake_context | 方案执行上下文（传输层、配置、会话等） |

### 返回值

`net::awaitable<handshake_result>` — 协程，返回执行结果（传输层、检测到的协议、预读数据、错误码）

### 调用（向下）

- execute_pipeline() — 执行方案管道
- execute_single() — 全部失败时执行 native 兜底
- find_scheme() — 查找 native 方案

### 被调用（向上）

- [[recognition/recognition|recognize]] — 识别流程中调用执行器

### 知识域

- [[recognition/result|analysis_result]] — ClientHello 分析结果

### 流程

1. 从 `analysis_result` 中获取候选方案名称列表
2. 候选为空时按注册顺序构建默认列表
3. 调用 `execute_pipeline()` 依次尝试候选方案
4. 全部失败则查找 "native" 方案并执行兜底
5. 返回最终结果

---

## 函数: execute()

> 源码: `include/prism/stealth/executor.hpp:54`
> 实现: `src/prism/stealth/executor.cpp:157`

### 功能

按候选列表执行方案管道。与 `execute_by_analysis` 类似，但直接接受候选方案名称列表，转发给 `execute_pipeline()`。

### 签名

```cpp
[[nodiscard]] auto execute(const memory::vector<memory::string> &candidates, handshake_context ctx) const
    -> net::awaitable<handshake_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| candidates | const memory::vector<memory::string>& | 候选方案名称列表 |
| ctx | handshake_context | 方案执行上下文 |

### 返回值

`net::awaitable<handshake_result>` — 协程，返回执行结果

### 调用（向下）

- execute_pipeline() — 执行方案管道

### 被调用（向上）

- execute_by_analysis() — 按分析结果驱动执行（间接，通过 execute_pipeline）

### 知识域

- [[stealth/scheme|handshake_context]] — 执行上下文结构体

---

## 函数: find_scheme()

> 源码: `include/prism/stealth/executor.hpp:60`
> 实现: `src/prism/stealth/executor.cpp:163`

### 功能

根据名称查找方案。从 `schemes_` 列表中线性查找指定名称的方案，未找到返回 nullptr。

### 签名

```cpp
[[nodiscard]] auto find_scheme(std::string_view name) const -> shared_scheme;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| name | std::string_view | 方案名称（如 "reality"、"native"） |

### 返回值

`shared_scheme` — 找到的方案智能指针，未找到返回 nullptr

### 调用（向下）

- 无（纯查找，调用 scheme->name() 比较）

### 被调用（向上）

- execute_pipeline() — 管道中查找候选方案
- execute_by_analysis() — 查找 native 兜底方案

### 知识域

- [[stealth/scheme|stealth_scheme]] — 方案基类接口

---

## 函数: execute_single()

> 源码: `include/prism/stealth/executor.hpp:62`
> 实现: `src/prism/stealth/executor.cpp:52`

### 功能

执行单个方案。调用方案的 `handshake()` 函数，完成后将方案名写入结果的 `scheme` 字段。

### 签名

```cpp
[[nodiscard]] static auto execute_single(shared_scheme scheme, handshake_context ctx)
    -> net::awaitable<handshake_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| scheme | shared_scheme | 要执行的方案智能指针 |
| ctx | handshake_context | 方案执行上下文 |

### 返回值

`net::awaitable<handshake_result>` — 协程，返回执行结果

### 调用（向下）

- [[stealth/scheme|stealth_scheme::handshake()]] — 方案握手函数

### 被调用（向上）

- execute_pipeline() — 管道中执行单个方案
- execute_by_analysis() — native 兜底执行

### 知识域

- [[stealth/scheme|handshake_result]] — 握手结果结构体

---

## 函数: pass_through()

> 源码: `include/prism/stealth/executor.hpp:65`
> 实现: `src/prism/stealth/executor.cpp:20`

### 功能

传递数据。将当前方案的执行结果传递给下一个方案。如果方案返回了新的 transport，则替换上下文中的 inbound；如果返回了 preread 数据，则用 [[pipeline/primitives|preview]] 包装 inbound 使下一方案能读取预读数据。

### 签名

```cpp
static auto pass_through(handshake_context &ctx, const handshake_result &res) -> void;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | handshake_context& | 方案执行上下文（会被修改） |
| res | const handshake_result& | 当前方案的执行结果 |

### 返回值

无

### 调用（向下）

- [[pipeline/primitives|preview]] — 用 preview 包装预读数据

### 被调用（向上）

- execute_pipeline() — 方案失败时传递数据给下一方案

### 知识域

- [[pipeline/primitives|preview]] — 预读数据包装器
- [[stealth/scheme|handshake_context]] — 执行上下文

---

## 函数: ensure_snapshot()

> 源码: `include/prism/stealth/executor.hpp:67`
> 实现: `src/prism/stealth/executor.cpp:32`

### 功能

确保快照。检查当前 inbound 是否已经是 [[channel/transport/snapshot|snapshot]] 包装，如果不是则包装一层 snapshot，使后续方案失败时可以 rewind 回退。

### 签名

```cpp
static auto ensure_snapshot(handshake_context &ctx) -> void;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | handshake_context& | 方案执行上下文（inbound 可能被包装） |

### 返回值

无

### 调用（向下）

- [[channel/transport/snapshot|make_snapshot]] — 创建 snapshot 包装

### 被调用（向上）

- execute_pipeline() — 每次执行方案前确保快照

### 知识域

- [[channel/transport/snapshot|snapshot]] — 传输层快照（支持 rewind）

---

## 函数: try_rewind()

> 源码: `include/prism/stealth/executor.hpp:69`
> 实现: `src/prism/stealth/executor.cpp:41`

### 功能

尝试回退。尝试将 `handshake_context` 中的 inbound 回退到快照位置。如果 inbound 不是 snapshot 或 snapshot 不可回退，返回 false。

### 签名

```cpp
static auto try_rewind(handshake_context &ctx) -> bool;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | handshake_context& | 方案执行上下文 |

### 返回值

`bool` — 回退成功返回 `true`，失败返回 `false`

### 调用（向下）

- [[channel/transport/snapshot|snapshot::can_rewind()]] — 检查是否可回退
- [[channel/transport/snapshot|snapshot::rewind()]] — 执行回退

### 被调用（向上）

- execute_pipeline() — 方案失败后尝试回退

### 知识域

- [[channel/transport/snapshot|snapshot]] — 传输层快照机制

---

## 函数: execute_pipeline()

> 源码: `include/prism/stealth/executor.hpp:71`
> 实现: `src/prism/stealth/executor.cpp:60`

### 功能

执行方案管道。按候选方案名称列表依次尝试执行，直到某个方案成功。对每个方案：先检查是否启用，然后 ensure_snapshot 快照，执行方案，根据结果决定继续或终止。返回 TLS 表示"不是我"尝试 rewind 后继续；成功则终止返回；错误时尝试 rewind，不能 rewind 则终止。

### 签名

```cpp
[[nodiscard]] auto execute_pipeline(const memory::vector<memory::string> &order, handshake_context ctx) const
    -> net::awaitable<handshake_result>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| order | const memory::vector<memory::string>& | 候选方案名称列表 |
| ctx | handshake_context | 方案执行上下文 |

### 返回值

`net::awaitable<handshake_result>` — 协程，返回执行结果

### 调用（向下）

- find_scheme() — 按名称查找方案
- execute_single() — 执行单个方案
- pass_through() — 传递数据给下一方案
- ensure_snapshot() — 确保快照
- try_rewind() — 尝试回退

### 被调用（向上）

- execute_by_analysis() — 按分析结果驱动执行
- execute() — 按候选列表执行

### 知识域

- [[stealth/scheme|stealth_scheme::active()]] — 方案启用检查
- [[stealth/scheme|handshake_result]] — 握手结果判断逻辑

### 流程

1. 遍历候选方案名称列表
2. 查找方案实例，跳过未找到或未启用的方案
3. 调用 `ensure_snapshot()` 包装快照
4. 调用 `execute_single()` 执行方案
5. **成功**: detected != tls && detected != unknown && transport 有效 → 返回结果
6. **返回 TLS**: 调用 `try_rewind()`，失败则 `pass_through()` 继续
7. **错误**: 调用 `try_rewind()`，成功则继续，失败则返回错误
8. **detected == unknown**: 尝试 rewind，失败则 pass_through
9. 全部失败返回 `fault::code::not_supported`
