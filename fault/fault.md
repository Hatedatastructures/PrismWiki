---
title: Fault 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [fault, error-handling, architecture, infrastructure]
related: [[exception/detail], [architecture], [crypto]]
module: fault
category: infrastructure
layer: error-handling
status: stable
dependencies: []
dependents:
  - exception
  - crypto
  - channel
  - pipeline
  - outbound
  - agent
---

# Fault 模块

## 1. 模块定位

Fault 模块是 Prism 的**错误码基础设施**，定义全局错误码枚举和统一的错误检查接口。它是双轨错误处理策略的热路径侧：热路径使用 `fault::code` 返回错误码，启动/致命错误使用异常（[[exception]]）。

**核心价值**：零分配、constexpr、热路径安全的错误传播机制。

**边界**：Fault 模块是 header-only 的纯基础设施，不依赖任何业务模块。

## 2. 核心功能

### 2.1 全局错误码枚举

`fault::code` 枚举覆盖系统所有错误情况，按功能分组：

| 分组 | 范围 | 示例 |
|------|------|------|
| 通用 | 0-10 | success, parse_error, eof, would_block |
| 网络 | 11-18 | timeout, tls_handshake_failed, dns_failed |
| 协议 | 19-25 | unsupported_command, blocked, bad_gateway |
| 安全 | 26-32 | ssl_cert_load_failed, config_parse_error |
| 系统 | 33-37 | connection_aborted, resource_unavailable, ipv6_disabled |
| 多路复用 | 38-44 | mux_session_error, mux_stream_error, mux_window_exceeded |
| SS2022 | 45-48 | crypto_error, invalid_psk, timestamp_expired, replay_detected |
| Reality | 49-57 | reality_auth_failed, reality_key_exchange_failed |
| UDP | 58-59 | udp_session_expired, packet_replay_detected |
| ECH | 60-63 | ech_payload_invalid, ech_decrypt_failed |

### 2.2 错误检查函数

```cpp
constexpr bool succeeded(code c);  // c == success
constexpr bool failed(code c);     // c != success
```

### 2.3 模板化统一检查

`handling.hpp` 提供对三种错误码类型的统一检查（`if constexpr` 编译时分发，零运行时开销）：
- `fault::code`
- `std::error_code`
- `boost::system::error_code`

### 2.4 错误码转换

`to_code()` 将 Boost.Asio 和 std 错误码映射到 `fault::code`：
- `boost::asio::error::eof` → `code::eof`
- `boost::asio::error::timed_out` → `code::timeout`
- 未映射错误 → `code::io_error`

### 2.5 标准库兼容

`compatible.hpp` 实现 `std::error_category` 和 `boost::system::error_category`，使 `fault::code` 可隐式转换为 `std::error_code` 和 `boost::system::error_code`。

## 3. 关键类和接口

### 3.1 `fault::code` — 全局错误码枚举（`code.hpp`）

```cpp
enum class code : int {
    success = 0,
    // 通用 (1-10)
    generic_error, parse_error, eof, would_block, protocol_error,
    bad_message, invalid_argument, not_supported, message_too_large, io_error,
    // 网络 (11-18)
    timeout, canceled, tls_handshake_failed, tls_shutdown_failed,
    auth_failed, dns_failed, upstream_unreachable, connection_refused,
    // 协议 (19-25)
    unsupported_command, unsupported_address, blocked, bad_gateway,
    host_unreachable, connection_reset, network_unreachable,
    // 安全 (26-32)
    ssl_cert_load_failed, ssl_key_load_failed, socks5_auth_negotiation_failed,
    file_open_failed, config_parse_error, port_already_in_use, certificate_verification_failed,
    // 系统 (33-37)
    connection_aborted, resource_unavailable, ttl_expired, forbidden, ipv6_disabled,
    // 多路复用 (38-44)
    mux_not_enabled, mux_session_error, mux_stream_error, mux_window_exceeded,
    mux_protocol_error, mux_connection_limit, mux_stream_limit,
    // SS2022 (45-48)
    crypto_error, invalid_psk, timestamp_expired, replay_detected,
    // Reality (49-57)
    reality_not_configured, reality_auth_failed, reality_sni_mismatch,
    reality_key_exchange_failed, reality_handshake_failed, reality_dest_unreachable,
    reality_certificate_error, reality_tls_record_error, reality_key_schedule_error,
    // UDP (58-59)
    udp_session_expired, packet_replay_detected,
    // ECH (60-63)
    ech_payload_invalid, ech_version_mismatch, ech_decrypt_failed, ech_config_mismatch,

    _count = 64
};
```

### 3.2 描述和检查函数（`code.hpp`）

```cpp
[[nodiscard]] constexpr std::string_view describe(code value) noexcept;  // 零分配，返回静态字面量
[[nodiscard]] constexpr bool succeeded(code c) noexcept;
[[nodiscard]] constexpr bool failed(code c) noexcept;
```

### 3.3 模板化错误检查（`handling.hpp`）

```cpp
template<typename ErrorCode> [[nodiscard]] constexpr bool succeeded(const ErrorCode &ec) noexcept;
template<typename ErrorCode> [[nodiscard]] constexpr bool failed(const ErrorCode &ec) noexcept;
[[nodiscard]] inline code to_code(const boost::system::error_code &ec) noexcept;
[[nodiscard]] inline code to_code(const std::error_code &ec) noexcept;
```

`succeeded<T>` 使用 `if constexpr` 编译时分发，其他类型触发 `static_assert` 编译错误。

### 3.4 标准库兼容层（`compatible.hpp`）

```cpp
namespace psm::fault {
    class fault_category : public std::error_category { ... };
    inline const std::error_category &category() noexcept;
    inline std::error_code make_error_code(code c) noexcept;
}
// std 和 boost 命名空间中均有 is_error_code_enum 特化
```

`cached_message()` 首次调用时预分配所有错误消息，后续调用返回引用（零分配）。

## 4. 错误码分类索引

```
fault::code
├── success (0)
├── 通用错误 (1-10)     — parse_error, eof, would_block, io_error
├── 网络错误 (11-18)    — timeout, tls_handshake_failed, dns_failed
├── 协议错误 (19-25)    — unsupported_command, blocked, bad_gateway
├── 安全错误 (26-32)    — ssl_cert_load_failed, config_parse_error
├── 系统错误 (33-37)    — connection_aborted, ipv6_disabled
├── Mux 错误 (38-44)    — mux_session_error, mux_window_exceeded
├── SS2022 错误 (45-48) — crypto_error, replay_detected
├── Reality 错误 (49-57) — reality_auth_failed, reality_key_exchange_failed
├── UDP 错误 (58-59)    — udp_session_expired, packet_replay_detected
└── ECH 错误 (60-63)    — ech_payload_invalid, ech_decrypt_failed
```

## 5. 文件清单

### 头文件（`include/prism/fault/`）

```
fault.hpp                # 聚合头文件
├── code.hpp             # 错误码枚举（code）、describe()、succeeded()、failed()
├── handling.hpp         # 模板化检查（succeeded<T>、failed<T>、to_code()）
└── compatible.hpp       # std/boost error_category、make_error_code、hash 特化
```

**注意**：Fault 模块是 header-only，无源文件。

## 6. 设计原理

- **热路径无异常**：网络 I/O、协议解析等热路径必须使用错误码返回值，禁止抛异常
- **零分配描述**：`describe()` 返回 `string_view` 指向编译期静态数据
- **双向兼容**：通过 `std::is_error_code_enum` 和 `boost::system::is_error_code_enum` 特化，`fault::code` 可隐式转换为两种标准错误码类型
- **to_code 桥接**：将 Boost.Asio 常见网络错误映射到内部错误码，隔离 Boost 依赖

## 相关页面

- [[exception]] — Exception 模块
- [[crypto]] — Crypto 模块（加密操作通过 fault::code 返回错误）
- [[architecture]] — 架构设计
