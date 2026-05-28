---
tags: [fault, code]
layer: core
module: fault
source: I:/code/Prism/include/prism/fault/code.hpp
title: fault::code
---

# fault::code

全局错误码枚举定义。热路径零异常原则：网络 I/O、协议解析等高频路径必须使用 `fault::code` 返回值，避免异常的栈展开开销。

## 核心接口

```cpp
enum class code : std::int32_t { ... };
[[nodiscard]] constexpr std::string_view describe(code value) noexcept;
[[nodiscard]] constexpr bool succeeded(code c) noexcept;
[[nodiscard]] constexpr bool failed(code c) noexcept;
```

## 错误码分组

| 范围 | 类别 | 关键枚举 |
|------|------|----------|
| 0 | 成功 | `success` |
| 1-10 | 通用 | `generic_error`, `parse_error`, `eof`, `would_block`, `bad_message`, `not_supported`, `oversized_msg` |
| 11-18 | 网络 | `timeout`, `canceled`, `tls_hsfail`, `tls_closefail`, `auth_failed`, `dns_failed`, `unreachable`, `connection_refused` |
| 19-25 | 协议 | `unsupported_command`, `unsupported_address`, `blocked`, `bad_gateway`, `host_noreply`, `connection_reset`, `net_noreply` |
| 26-37 | 安全/系统 | `certfail`, `keyfail`, `socks5_authfail`, `file_openfail`, `config_err`, `port_busy`, `verifyfail`, `connection_aborted`, `resource_unavailable`, `forbidden`, `ipv6_disabled` |
| 38-44 | 多路复用 | `mux_disabled`, `sessfail`, `streamfail`, `mux_overflow`, `protoerr`, `connlimit`, `streamcap` |
| 45-48 | SS2022 | `crypto_error`, `invalid_psk`, `timestamp_expired`, `replay_detected` |
| 49-57 | Reality | `unset`, `unauth`, `badsni`, `kexfail`, `hsfail`, `unreach`, `st_certfail`, `recorderr`, `kdferr` |
| 58-59 | UDP | `udp_expired`, `pkt_replay` |
| 60-63 | ECH | `badpayload`, `badver`, `decfail`, `badcfg` |

完整枚举值列表见 `include/prism/fault/code.hpp:30-168`。

## 设计决策

### 为什么错误码用缩写命名（如 `tls_hsfail` 而非 `tls_handshake_failed`？

遵循项目标识符命名规范（Rule 2.1）：不超过 2 个词，最多 1 个下划线。`describe()` 返回完整名称字符串供日志使用，代码中用短名保持可读性。

**后果**: 新增错误码必须同时更新枚举定义和 `describe()` 的 switch 语句，两者需一一对应。

### 为什么 describe() 返回 string_view 而非 string？

返回 `std::string_view` 指向 switch 分支中的字符串字面量，零内存分配。热路径日志（可能每秒数千次）避免 `std::string` 堆分配开销。

**后果**: `describe()` 返回值生命周期与程序相同，可安全存储但不可修改。

## 使用场景

| 场景 | 示例 |
|------|------|
| 异步 I/O 返回值 | `auto ec = co_await read_frame(); if (fault::failed(ec)) ...` |
| 日志 | `trace::error("failed: {}", fault::describe(ec));` |
| 错误传播 | `return ec;`（透传，不要包装为 `generic_error`） |

## 引用关系

### 被引用

- [[core/fault/overview|Fault 模块]]：总览
- [[core/fault/handling|handling]]：统一检查接口
- [[core/fault/compatible|compatible]]：标准库兼容
- [[core/exception/deviant|exception::deviant]]：异常基类携带错误码
