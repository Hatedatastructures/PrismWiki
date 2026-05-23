---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/request.hpp
title: Reality Request
---

# Reality Request

TLS ClientHello 解析器。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/request.hpp`

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