---
title: "Fault 模块"
module: "fault"
type: module
tags: [fault, 错误码, error, handling, compatible]
created: 2026-05-15
updated: 2026-05-15
related:
  - exception/deviant
  - agent/config
  - channel/transport/transmission
  - resolve/router
  - recognition/probe
---

# Fault 模块

> 模块: [[fault|fault]]

## 概述

Fault 模块定义错误码枚举和双轨错误处理机制。提供 `fault::code` 枚举、`std::error_code` 适配和统一错误检查接口。遵循热路径无异常原则，网络 I/O、协议解析等热路径必须使用错误码返回值进行流控，异常仅用于启动阶段或致命错误。

---

## code.hpp — 错误码枚举

> 源码: `include/prism/fault/code.hpp`
> 模块: [[fault|fault]]

### 概述

定义 `fault::code` 枚举及基础辅助函数。错误码按功能分组：通用(1-10)、网络(11-18)、协议(19-25)、安全(26-32)、系统(33-36)、多路复用(38-44)、SS2022(45-48)、Reality(49-57)、UDP(58-59)、ECH(60-63)。

### 命名空间

`psm::fault`

### 枚举: code

- **功能说明**: 全局错误码。表示系统运行时可能出现的所有错误情况，零值表示成功，非零值表示各类错误。
- **签名**: `enum class code : int`
- **参数**: 无

| 值 | 整数 | 说明 |
|----|------|------|
| `success` | 0 | 操作成功 |
| `generic_error` | 1 | 通用错误 |
| `parse_error` | 2 | 解析错误 |
| `eof` | 3 | 到达文件末尾 |
| `would_block` | 4 | 操作将阻塞 |
| `protocol_error` | 5 | 协议错误 |
| `bad_message` | 6 | 消息格式错误 |
| `invalid_argument` | 7 | 无效参数 |
| `not_supported` | 8 | 不支持的操作 |
| `message_too_large` | 9 | 消息过大 |
| `io_error` | 10 | I/O 错误 |
| `timeout` | 11 | 操作超时 |
| `canceled` | 12 | 操作被取消 |
| `tls_handshake_failed` | 13 | TLS 握手失败 |
| `tls_shutdown_failed` | 14 | TLS 关闭失败 |
| `auth_failed` | 15 | 认证失败 |
| `dns_failed` | 16 | DNS 解析失败 |
| `upstream_unreachable` | 17 | 上游服务器不可达 |
| `connection_refused` | 18 | 连接被拒绝 |
| `unsupported_command` | 19 | 不支持的命令 |
| `unsupported_address` | 20 | 不支持的地址类型 |
| `blocked` | 21 | 请求被阻止 |
| `bad_gateway` | 22 | 网关错误 |
| `host_unreachable` | 23 | 主机不可达 |
| `connection_reset` | 24 | 连接被重置 |
| `network_unreachable` | 25 | 网络不可达 |
| `ssl_cert_load_failed` | 26 | SSL 证书加载失败 |
| `ssl_key_load_failed` | 27 | SSL 密钥加载失败 |
| `socks5_auth_negotiation_failed` | 28 | SOCKS5 认证协商失败 |
| `file_open_failed` | 29 | 文件打开失败 |
| `config_parse_error` | 30 | 配置解析错误 |
| `port_already_in_use` | 31 | 端口已被占用 |
| `certificate_verification_failed` | 32 | 证书验证失败 |
| `connection_aborted` | 33 | 连接被中止 |
| `resource_unavailable` | 34 | 资源不可用 |
| `ttl_expired` | 35 | TTL 已过期 |
| `forbidden` | 36 | 禁止访问 |
| `ipv6_disabled` | 37 | IPv6 被禁用 |
| `mux_not_enabled` | 38 | Mux 未启用 |
| `mux_session_error` | 39 | Mux 会话错误 |
| `mux_stream_error` | 40 | Mux 流错误 |
| `mux_window_exceeded` | 41 | Mux 窗口超限 |
| `mux_protocol_error` | 42 | Mux 协议错误 |
| `mux_connection_limit` | 43 | Mux 连接数限制 |
| `mux_stream_limit` | 44 | Mux 流数限制 |
| `crypto_error` | 45 | AEAD 加密/解密失败 |
| `invalid_psk` | 46 | PSK 长度或 base64 无效 |
| `timestamp_expired` | 47 | 客户端时间戳超出有效窗口 |
| `replay_detected` | 48 | Salt 重放检测 |
| `reality_not_configured` | 49 | Reality 未配置 |
| `reality_auth_failed` | 50 | Reality 认证失败 |
| `reality_sni_mismatch` | 51 | SNI 不在 server_names 中 |
| `reality_key_exchange_failed` | 52 | X25519 密钥交换失败 |
| `reality_handshake_failed` | 53 | Reality TLS 握手失败 |
| `reality_dest_unreachable` | 54 | 回退目标服务器不可达 |
| `reality_certificate_error` | 55 | 证书获取/处理失败 |
| `reality_tls_record_error` | 56 | TLS 记录解析/生成错误 |
| `reality_key_schedule_error` | 57 | TLS 1.3 密钥调度错误 |
| `udp_session_expired` | 58 | UDP 会话已过期 |
| `packet_replay_detected` | 59 | UDP PacketID 重放检测 |
| `ech_payload_invalid` | 60 | ECH payload 无效 |
| `ech_version_mismatch` | 61 | ECH version 不匹配 |
| `ech_decrypt_failed` | 62 | ECH 解密失败 |
| `ech_config_mismatch` | 63 | ECH config_id 不匹配 |
| `_count` | 64 | 错误码总数（仅供内部使用） |

- **返回值**: 不适用（枚举定义）
- **调用（向下）**: 无
- **被调用（向上）**: 全模块热路径错误返回
- **知识域**: 错误码分类体系

### 函数: describe()

- **功能说明**: 获取错误码的零分配描述。将错误码转换为人类可读的字符串描述，返回的字符串视图指向静态存储期数据，可安全用于日志和诊断。
- **签名**: `[[nodiscard]] constexpr std::string_view describe(code value) noexcept`
- **参数**: `value` — 错误码枚举值
- **返回值**: `std::string_view` — 错误描述字符串视图，生命周期与程序相同。未知错误码返回 `"unknown"`。
- **调用（向下）**: 无（纯 switch 映射）
- **被调用（向上）**: `compatible::cached_message()`、`stealth::executor`、`recognition::recognize()`、`pipeline::primitives::dial()`、`outbound::direct`
- **知识域**: 错误码可读化

### 函数: succeeded() [code.hpp]

- **功能说明**: 检查错误码是否表示成功。语义等价于 `c == code::success`。
- **签名**: `[[nodiscard]] constexpr bool succeeded(code c) noexcept`
- **参数**: `c` — 错误码枚举值
- **返回值**: `bool` — success 返回 true，否则返回 false
- **调用（向下）**: 无
- **被调用（向上）**: `handling::succeeded()` 模板特化、协议处理流程
- **知识域**: 错误码语义检查

### 函数: failed() [code.hpp]

- **功能说明**: 检查错误码是否表示失败。`succeeded()` 的互补函数，语义等价于 `c != code::success`。
- **签名**: `[[nodiscard]] constexpr bool failed(code c) noexcept`
- **参数**: `c` — 错误码枚举值
- **返回值**: `bool` — 非 success 返回 true，否则返回 false
- **调用（向下）**: `succeeded()`
- **被调用（向上）**: `handling::failed()` 模板特化、`stealth::executor`、`recognition::recognize()`、`pipeline::primitives::dial()`、`outbound::direct`
- **知识域**: 错误码语义检查

---

## compatible.hpp — std::error_code 适配

> 源码: `include/prism/fault/compatible.hpp`
> 模块: [[fault|fault]]

### 概述

`fault::code` 与 `std::error_code` 和 `boost::system::error_code` 的双向兼容性实现。包括错误分类、哈希支持和隐式转换特化。实现 `std::error_category` 使 `fault::code` 可以与 Boost.Asio 的错误处理机制无缝集成。

### 函数: cached_message()

- **功能说明**: 获取缓存的错误消息。返回预分配的错误消息引用，首次调用时分配并缓存，后续调用直接返回引用，无内存分配。
- **签名**: `[[nodiscard]] inline const std::string &cached_message(code c) noexcept`
- **参数**: `c` — 错误码枚举值
- **返回值**: `const std::string&` — 错误消息字符串的常量引用，生命周期与程序相同
- **调用（向下）**: `describe()`
- **被调用（向上）**: `fault_category::message()`（std 和 boost 两个版本）
- **知识域**: 错误消息缓存策略

### 类: fault_category [std]

- **功能说明**: `std::error_code` 错误分类。实现 `std::error_category` 接口，为 `fault::code` 提供标准库错误分类支持。
- **签名**: `class fault_category : public std::error_category`

#### fault_category::name() [std]

- **功能说明**: 获取分类名称。
- **签名**: `const char *name() const noexcept override`
- **参数**: 无
- **返回值**: `const char*` — 分类名称字符串 `"psm::fault"`
- **调用（向下）**: 无
- **被调用（向上）**: `std::error_code` 框架
- **知识域**: error_category 接口实现

#### fault_category::message() [std]

- **功能说明**: 获取错误码对应的消息。
- **签名**: `std::string message(int c) const override`
- **参数**: `c` — 错误码整数值
- **返回值**: `std::string` — 错误消息字符串
- **调用（向下）**: `cached_message()`
- **被调用（向上）**: `std::error_code` 框架
- **知识域**: error_category 接口实现

### 函数: category() [std]

- **功能说明**: 获取状态分类单例。首次调用时构造单例，C++11 保证线程安全。
- **签名**: `inline const std::error_category &category() noexcept`
- **参数**: 无
- **返回值**: `const std::error_category&` — fault_category 单例引用
- **调用（向下）**: `fault_category` 构造
- **被调用（向上）**: `make_error_code()`、`handling::to_code(std::error_code)`
- **知识域**: error_category 单例模式

### 函数: make_error_code() [std]

- **功能说明**: 将 `fault::code` 枚举值转换为 `std::error_code`。配合 `is_error_code_enum` 特化支持隐式转换。
- **签名**: `inline std::error_code make_error_code(code c) noexcept`
- **参数**: `c` — 自定义错误码枚举值
- **返回值**: `std::error_code` — 对应的标准错误码对象
- **调用（向下）**: `category()`
- **被调用（向上）**: `exception::deviant` 构造、`exception::network` 构造、`exception::protocol` 构造、`exception::security` 构造
- **知识域**: error_code 转换

### 特化: std::is_error_code_enum\<psm::fault::code\>

- **功能说明**: 标记 `fault::code` 为错误码枚举，启用与 `std::error_code` 的隐式转换。
- **签名**: `template<> struct is_error_code_enum<psm::fault::code> : std::true_type`
- **参数**: 无
- **返回值**: 不适用（类型特化）
- **调用（向下）**: 无
- **被调用（向上）**: `std::error_code` 隐式转换机制
- **知识域**: 标准库枚举适配

### 特化: std::hash\<psm::fault::code\>

- **功能说明**: 为 `fault::code` 提供 `std::hash` 特化，使其可用于无序容器。哈希实现委托给 `std::hash<int>`。
- **签名**: `template<> struct hash<psm::fault::code>`
- **参数**: `c` — 错误码枚举值
- **返回值**: `size_t` — 哈希值
- **调用（向下）**: `std::hash<int>{}()`
- **被调用（向上）**: 无序容器（`unordered_map`、`unordered_set`）
- **知识域**: 错误码哈希支持

### 类: fault_category [boost]

- **功能说明**: Boost 错误码分类。实现 `boost::system::error_category` 接口，与标准库版本保持功能对等。
- **签名**: `class fault_category final : public boost::system::error_category`
- **参数**: 无
- **返回值**: 不适用
- **调用（向下）**: `cached_message()`
- **被调用（向上）**: `boost::system::error_code` 框架
- **知识域**: Boost error_category 接口实现

### 函数: category() [boost]

- **功能说明**: 获取 Boost 状态分类单例。首次调用时构造单例，C++11 保证线程安全。
- **签名**: `inline const boost::system::error_category &category() noexcept`
- **参数**: 无
- **返回值**: `const boost::system::error_category&` — fault_category 单例引用
- **调用（向下）**: `fault_category` 构造
- **被调用（向上）**: `make_error_code()`（boost 版本）
- **知识域**: Boost error_category 单例模式

### 函数: make_error_code() [boost]

- **功能说明**: 将 `fault::code` 枚举值转换为 `boost::system::error_code`。配合特化支持隐式转换。
- **签名**: `inline boost::system::error_code make_error_code(psm::fault::code c) noexcept`
- **参数**: `c` — 自定义错误码枚举值
- **返回值**: `boost::system::error_code` — 对应的 Boost 错误码对象
- **调用（向下）**: `category()`（boost 版本）
- **被调用（向上）**: Boost.Asio 错误处理机制
- **知识域**: Boost error_code 转换

### 特化: boost::system::is_error_code_enum\<psm::fault::code\>

- **功能说明**: 标记 `fault::code` 为 Boost 错误码枚举，启用与 `boost::system::error_code` 的隐式转换。
- **签名**: `template<> struct is_error_code_enum<psm::fault::code> : std::true_type`
- **参数**: 无
- **返回值**: 不适用（类型特化）
- **调用（向下）**: 无
- **被调用（向上）**: `boost::system::error_code` 隐式转换机制
- **知识域**: Boost 枚举适配

---

## handling.hpp — 错误检查适配层

> 源码: `include/prism/fault/handling.hpp`
> 模块: [[fault|fault]]

### 概述

极简错误码检查适配层。提供对 `fault::code`、`std::error_code` 和 `boost::system::error_code` 的统一错误检查接口。所有函数均为 constexpr 和 noexcept，无动态分配，专为热路径设计。

### 函数: succeeded() [handling.hpp]

- **功能说明**: 检查错误码是否表示成功。使用 `if constexpr` 实现编译时类型分发，消除运行时类型检查开销。支持 `fault::code`、`std::error_code`、`boost::system::error_code` 三种类型。
- **签名**: `template <typename ErrorCode> [[nodiscard]] constexpr bool succeeded(const ErrorCode &ec) noexcept`
- **参数**: `ec` — 错误码对象的常量引用
- **返回值**: `bool` — true 表示操作成功，false 表示操作失败
- **调用（向下）**: `code::succeeded()`（fault::code 特化时）
- **被调用（向上）**: 热路径中的统一错误检查
- **知识域**: 统一错误码检查

### 函数: failed() [handling.hpp]

- **功能说明**: 检查错误码是否表示失败。`succeeded()` 的互补函数，语义等价于 `!succeeded(ec)`。
- **签名**: `template <typename ErrorCode> [[nodiscard]] constexpr bool failed(const ErrorCode &ec) noexcept`
- **参数**: `ec` — 错误码对象的常量引用
- **返回值**: `bool` — true 表示操作失败，false 表示操作成功
- **调用（向下）**: `succeeded()`
- **被调用（向上）**: 热路径中的统一错误检查
- **知识域**: 统一错误码检查

### 函数: to_code(boost::system::error_code)

- **功能说明**: 将 Boost 错误码转换为 `fault::code`。映射常见 Boost.Asio 网络错误到对应的 fault 错误码，未映射的错误返回 `io_error`。
- **签名**: `[[nodiscard]] inline code to_code(const boost::system::error_code &ec) noexcept`
- **参数**: `ec` — Boost 系统错误码
- **返回值**: `code` — 对应的内部错误码
- **调用（向下）**: Boost.Asio 错误比较
- **被调用（向上）**: `pipeline::primitives::recover()`、`vless::relay`
- **知识域**: Boost 错误码映射

### 函数: to_code(std::error_code)

- **功能说明**: 将 std 错误码转换为 `fault::code`。映射常见 `std::errc` 错误到对应的 fault 错误码，未映射的错误返回 `io_error`。
- **签名**: `[[nodiscard]] inline code to_code(const std::error_code &ec) noexcept`
- **参数**: `ec` — C++ 标准库错误码
- **返回值**: `code` — 对应的内部错误码
- **调用（向下）**: `category()`、`std::errc` 比较
- **被调用（向上）**: 标准库错误码到内部错误码的转换
- **知识域**: std 错误码映射

---

## fault 概述

Fault 模块的详细设计文档。双轨错误处理策略：热路径使用 `fault::code` 枚举返回错误码，启动/致命错误使用异常层次结构。通过 `compatible.hpp` 实现与 `std::error_code` 和 `boost::system::error_code` 的双向兼容。

## 相关页面

- [[exception/deviant|exception]] — 异常层次结构，启动阶段使用
- [[agent/config|config]] — 配置中的错误处理参数
- [[channel/transport/transmission|transmission]] — 传输层使用 fault::code 返回 I/O 错误
- [[resolve/router|router]] — DNS 路由使用 fault::code 返回解析错误
- [[recognition/probe|probe]] — 协议探测使用 fault::code 返回探测错误
