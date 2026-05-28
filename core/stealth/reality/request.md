---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/request.hpp
title: Reality Request
tags:
  - stealth
  - reality
  - ClientHello
  - parser
---

# Reality Request

TLS ClientHello 解析器。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/request.hpp`

## 设计决策（WHY）

### 为什么 `raw_message` 必须保留

TLS 1.3 密钥调度使用 transcript hash（所有握手消息的 SHA256 哈希）。ClientHello 是 transcript 的第一部分，后续的 `keygen::derive_handshake_keys` 需要完整的 ClientHello 字节。如果解析后丢弃原始字节，就无法正确计算 transcript hash。

### 为什么 `client_hello_info` 和 `client_hello_features` 是两个不同的结构

`client_hello_features` 来自 `recognition` 模块，用于特征分析和方案检测。`client_hello_info` 用于 Reality 握手流程，包含不同的字段集（如 `raw_message`、`client_public_key`）。两者字段有重叠但职责不同。源码中有一个 TODO 标记："统一 client_hello_features 与 client_hello_info 类型，消除此转换"。

### 为什么 X25519 公钥是 32 字节固定数组

X25519 密钥交换的公钥固定 32 字节。使用 `std::array<uint8_t, 32>` 而非 `vector` 避免堆分配，且长度保证在编译期。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| Content Type 必须是 0x16 (Handshake) | TLS 规范 | 非 Handshake 记录被拒绝 |
| Handshake Type 必须是 0x01 (ClientHello) | TLS 规范 | 非 ClientHello 被拒绝 |
| session_id 最长 32 字节 | TLS 规范 | 超长 session_id 导致解析失败 |
| 解析是同步操作 | 无 I/O | 纯字节解析，可在任何上下文调用 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| TLS 记录头读取失败 | 连接中断 | `handshake` 返回 `io_error` |
| Content Type 不是 0x16 | 非 TLS 握手数据 | 解析失败，触发 fallback |
| Handshake Type 不是 0x01 | ServerHello 误读 | 解析失败 |
| 缺少 key_share 扩展 | 非 Reality 客户端 | `has_client_public_key=false`，认证失败 |
| SNI 扩展缺失 | 无 server_name | `server_name` 为空 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `request` → `transport::transmission` | 依赖 | `read_tls_record` 从传输层读取 |
| `handshake` → `request` | 调用 | Stage 1 读取和解析 ClientHello |
| `auth` → `client_hello_info` | 使用 | 认证时使用 SNI、session_id、public_key |
| `response` → `client_hello_info` | 使用 | 构造 ServerHello 时使用 client_hello 信息 |
| `keygen` → `raw_message` | 使用 | 作为 transcript hash 输入 |

## 解析结果

### client_hello_info

```cpp
struct client_hello_info
{
    memory::vector<std::uint8_t> raw_message;         // 完整 ClientHello
    std::array<std::uint8_t, 32> random{};            // 客户端随机数
    memory::vector<std::uint8_t> session_id;          // session_id
    memory::string server_name;                       // SNI
    std::array<std::uint8_t, 32> client_public_key{}; // key_share X25519 公钥
    bool has_client_public_key = false;               // 是否有客户端公钥
    memory::vector<std::uint16_t> supported_versions; // 支持的 TLS 版本
};
```

| 字段 | 用途 |
|------|------|
| `raw_message` | transcript hash 计算 |
| `random` | 客户端随机数（32 字节） |
| `session_id` | 包含 Reality short_id 和认证数据 |
| `server_name` | SNI 匹配验证 |
| `client_public_key` | X25519 密钥交换 |
| `supported_versions` | TLS 版本协商 |

## 函数接口

### read_tls_record

```cpp
auto read_tls_record(transport::transmission &transport)
    -> net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>
```

从传输层读取完整 TLS 记录。

返回完整的 TLS 记录（含 5 字节 record header）。

### parse_client_hello

```cpp
[[nodiscard]] auto parse_client_hello(std::span<const std::uint8_t> raw_tls_record)
    -> std::pair<fault::code, client_hello_info>
```

解析 ClientHello 提取关键字段。

## TLS 记录结构

```
TLS Record (5 + N bytes):
[ContentType: 1B][Version: 2B][Length: 2B][Payload: N bytes]

ClientHello Payload:
[HandshakeType: 1B][Length: 3B][Version: 2B][Random: 32B][SessionID: 1+len][CipherSuites: 2+list][CompressionMethods: 1+list][Extensions: 2+list]

Extensions:
- server_name (0x0000): SNI
- key_share (0x0033): X25519 公钥
- supported_versions (0x002B): TLS 版本列表
```

## 解析流程

1. 读取 TLS 记录头（5 字节）
2. 验证 Content Type = 0x16 (Handshake)
3. 验证 Handshake Type = 0x01 (ClientHello)
4. 解析随机数（32 字节）
5. 解析 session_id（1+len）
6. 跳过 cipher_suites 和 compression_methods
7. 遍历 extensions 提取：
   - SNI (server_name)
   - X25519 公钥 (key_share)
   - 支持版本 (supported_versions)

## 特性标记

Reality 客户端特征：

- `session_id[0] == 0x01, session_id[1] == 0x08, session_id[2] == 0x02`：Reality 独占标记
- `key_share` 包含 X25519 (named_group = 0x001D)
- `session_id` 长度 = 32 字节（完整）

## 调用链

- [[handshake]] ← 调用解析
- [[auth]] ← 使用解析结果
- [[constants]] ← TLS 常量
- [[response]] ← 使用 raw_message