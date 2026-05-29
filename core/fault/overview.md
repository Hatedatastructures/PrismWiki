---
tags: [fault, overview]
layer: core
module: fault
source: include/prism/fault/code.hpp
  - include/prism/fault/code.hpp
  - include/prism/fault/handling.hpp
  - include/prism/fault/compatible.hpp
title: Fault 模块
---

# Fault 模块

Fault 模块提供统一的错误码系统，遵循热路径无异常原则。

## 设计决策

### 为什么使用双轨策略（错误码 vs 异常）？

热路径（网络 I/O、协议解析、TLS 握手）使用 `fault::code` 错误码返回值，避免异常的栈展开开销和不可预测的延迟。冷路径（启动配置解析、参数校验）使用异常层次 `exception::deviant`，因为错误是致命的、不可恢复的，异常自动展开调用栈便于清理。

**后果**: 热路径函数必须返回 `fault::code` 或包含它的 `pair`/`struct`。调用方必须显式检查返回值，否则错误被静默丢弃。

### 为什么 describe() 返回 string_view 而非 string？

`describe()` 返回 `std::string_view` 指向静态字符串字面量，零内存分配。在热路径错误日志中（可能每秒数千次），避免 `std::string` 构造的堆分配开销。

**后果**: `describe()` 返回的 `string_view` 生命周期与程序相同，可安全存储。

## 模块组成

| 组件 | 说明 | 源码 |
|------|------|------|
| [[core/fault/code\|code]] | 错误码枚举定义 | `prism/fault/code.hpp` |
| [[core/fault/handling\|handling]] | 错误检查适配层 | `prism/fault/handling.hpp` |
| [[core/fault/compatible\|compatible]] | 标准库兼容性 | `prism/fault/compatible.hpp` |

## 错误码分组

| 范围 | 类别 | 示例 |
|------|------|------|
| 0 | 成功 | `success` |
| 1-10 | 通用 | `generic_error`, `parse_error`, `eof` |
| 11-18 | 网络 | `timeout`, `canceled`, `tls_handshake_failed` |
| 19-25 | 协议 | `unsupported_command`, `bad_gateway` |
| 26-36 | 安全/系统 | `ssl_cert_load_failed`, `file_open_failed` |
| 38-44 | 多路复用 | `mux_session_error`, `mux_stream_limit` |
| 45-48 | SS2022 | `crypto_error`, `replay_detected` |
| 49-57 | Reality | `reality_auth_failed`, `reality_key_exchange_failed` |
| 58-59 | UDP | `udp_session_expired` |
| 60-63 | ECH | `ech_decrypt_failed` |

## 核心接口

```cpp
namespace psm::fault {
    enum class code : int { ... };
    constexpr std::string_view describe(code value) noexcept;
    constexpr bool succeeded(code c) noexcept;
    constexpr bool failed(code c) noexcept;
}
```

## 错误码 vs 异常决策

| 场景 | 使用 | 原因 |
|------|------|------|
| 网络 I/O | `fault::code` | 预期内，高频 |
| 协议解析 | `fault::code` | 预期内，高频 |
| DNS 查询 | `fault::code` | 预期内，中频 |
| TLS 握手 | `fault::code` | 预期内，中频 |
| 配置文件解析 | exception | 启动阶段，致命 |
| 参数校验 (公共 API) | exception | 编程错误 |
| 不可恢复的内部状态 | exception | 逻辑错误 |

## 错误恢复策略

| 错误码类别 | 恢复策略 | 说明 |
|-----------|---------|------|
| `timeout`, `would_block` | 重试（指数退避） | 临时性错误 |
| `eof` | 关闭连接，清理状态 | 连接关闭 |
| `connection_refused` | 切换备用节点 | 当前节点不可用 |
| `auth_failed`, `blocked` | 终止连接 | 不可恢复 |
| `crypto_error`, `replay_detected` | 关闭连接，安全告警 | 可能的安全事件 |

## 相关模块

- [[core/exception/overview|Exception 模块]]：异常系统（仅启动/致命路径）
- [[core/connect/dial/dial|Dial]]：连接层错误码消费者
- [[core/stealth/scheme|Stealth]]：伪装方案错误码消费者
## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| header-only | 所有函数必须为 `constexpr` 或 `inline`，禁止 .cpp 文件 | 链接错误或 ODR 违规 | `code.hpp`, `handling.hpp`, `compatible.hpp` |
| 零分配 | `describe()` 返回 `string_view` 指向静态字面量，禁止堆分配 | 热路径日志引入 GC 压力 | `code.hpp` describe() |
| noexcept 保证 | 所有公共 API 标记 `noexcept`，不可抛异常 | 调用方异常安全假设被打破 | 所有公共函数签名 |
| 枚举值不可变 | 已发布的 `code` 值不可更改，仅可追加 | ABI 破坏，二进制兼容性丧失 | `code.hpp` enum class code |
| _count 内部专用 | `code::_count` 仅用于内部边界检查，不可用于错误处理 | 越界枚举值传入 describe() 返回 "unknown" | `code.hpp` 注释 |
| 双轨分流 | 热路径禁止 `try/catch`，冷路径必须使用异常层次 | 混用导致性能退化或错误丢失 | 项目架构约定 |

## 故障场景

### 1. 未知错误码传入 describe()

**触发**: 新增枚举值但未更新 `describe()` 的 switch 语句（编译器不会警告 default 分支遗漏）。

**表现**: 返回 `"unknown"`，日志中丢失具体错误信息，排查困难。

**恢复**: 无法自动恢复，需代码修复。CI 应检查 `describe()` 覆盖所有枚举值。

### 2. to_code() 映射不完整

**触发**: Boost.Asio 或标准库新增错误类型未被 `to_code()` 覆盖。

**表现**: 未映射的错误回退为 `code::io_error`，原始错误语义丢失。例如新的 TLS alert 被归类为通用 I/O 错误。

**恢复**: 日志中记录原始 `error_code` 用于事后排查，但运行时无法区分具体原因。

### 3. cached_message() 静态析构竞争

**触发**: 在程序退出阶段（静态析构期间）调用 `cached_message()` 或 `category()`。

**表现**: 静态局部变量 `messages` 可能已被销毁，返回悬垂引用，导致 UB（崩溃或乱码）。

**恢复**: 无法恢复。文档已标注禁止在静态析构阶段使用。

### 4. 错误码静默丢弃

**触发**: 调用方不检查 `fault::code` 返回值（C++ 不强制 `[[nodiscard]]` 检查）。

**表现**: 错误被忽略，后续逻辑基于成功假设执行，可能导致数据损坏或连接泄漏。

**恢复**: 编译期添加 `[[nodiscard]]` 到返回 `fault::code` 的函数可部分缓解，但无法根治。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| 错误码传播 | fault → 所有热路径模块 | transport/connect/stealth/protocol 等模块的异步操作返回 `fault::code` |
| 异常委托 | fault → exception | 冷路径通过 `exception::deviant` 层次报告致命错误，与 fault 互补 |
| Boost 互操作 | fault ↔ boost::asio | `to_code()` 将 `boost::system::error_code` 转为 `fault::code`，`compatible.hpp` 支持反向隐式转换 |
| 标准库互操作 | fault ↔ std | `is_error_code_enum` 特化启用 `fault::code` 到 `std::error_code` 隐式转换 |
| 日志消费 | fault → trace | `describe()` 输出被 spdlog 日志系统消费，格式变更影响日志可读性 |
| Dial 层消费 | connect → fault | `dial()` 返回 `fault::code`，router/racer 根据错误码决定重试或切换节点 |

## 变更敏感度

### 对外影响

| 变更类型 | 影响范围 | 破坏性 |
|----------|----------|--------|
| 新增枚举值 | 所有包含 `switch(code)` 的代码需更新 default 分支 | 低（追加式） |
| 修改已有枚举值 | 二进制兼容性破坏，所有依赖模块需重编译 | **高** |
| `describe()` 签名变更 | 所有日志/格式化代码需适配 | **高** |
| `to_code()` 映射表变更 | 错误传播语义改变，可能影响重试/故障切换逻辑 | 中 |
| 删除枚举值 | 编译错误，所有引用处需修改 | **高** |

### 对内影响

| 变更类型 | 影响范围 | 风险 |
|----------|----------|------|
| `compatible.hpp` 结构调整 | 影响 `std::hash` 和 `is_error_code_enum` 特化，可能触发 ODR 问题 | 中 |
| `handling.hpp` 新增模板重载 | 需确保 `static_assert` 覆盖所有不支持的类型 | 低 |
| `cached_message()` 缓存策略变更 | 首次调用延迟和内存占用可能改变 | 低 |
