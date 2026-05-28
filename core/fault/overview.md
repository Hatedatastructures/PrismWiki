---
tags: [fault, overview]
layer: core
module: fault
source:
  - I:/code/Prism/include/prism/fault/code.hpp
  - I:/code/Prism/include/prism/fault/handling.hpp
  - I:/code/Prism/include/prism/fault/compatible.hpp
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