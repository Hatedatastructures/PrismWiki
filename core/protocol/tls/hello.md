---
title: TLS ClientHello 解析器
layer: core
source:
  - include/prism/protocol/tls/hello.hpp
  - src/prism/protocol/tls/hello.cpp
tags: [protocol, tls, clienthello, parser]
---

# TLS ClientHello 解析器

> 源码: `include/prism/protocol/tls/hello.hpp` | 实现: `src/prism/protocol/tls/hello.cpp`

从 TLS 记录帧中解析 ClientHello 消息，提取 SNI、session_id、X25519 公钥等关键特征，供 recognition 和 stealth 模块使用。

## 核心类

### `psm::tls::client_hello`

不可变的 ClientHello 解析结果，提供只读字段访问。

**解析入口**:

| 方法 | 输入 | 说明 |
|------|------|------|
| `from(record)` | `tls::record` | 从 TLS 记录帧解析 |
| `from_bytes(span)` | 原始字节 | 从裸字节解析 |

**字段访问**:

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `sni()` | `string_view` | Server Name Indication |
| `session_id()` | `span<const uint8_t>` | 会话标识 |
| `has_x25519()` | `bool` | 是否包含 X25519 key_share |
| `x25519_key()` | `array<uint8_t,32>` | X25519 公钥 |
| `versions()` | `span<const uint16_t>` | 支持的 TLS 版本 |
| `random()` | `array<uint8_t,32>` | 客户端随机数 |
| `raw_msg()` | `span<const uint8_t>` | 原始握手消息（不含 record header） |
| `raw_record()` | `span<const std::byte>` | 原始 TLS 记录（含 record header） |

**向后兼容**: `to_features()` 将解析结果转换为 `protocol::tls::hello_features` 结构体。详见 [[core/protocol/tls/types|TLS 类型定义]]。

## 解析流程

```
tls::record → 提取 payload → 验证 handshake_type=0x01
       │
       ▼
解析固定字段: version(2B) + random(32B) + session_id(0-32B)
       │
       ▼
解析 cipher_suites (2B 每项)
       │
       ▼
解析 extensions:
  - 0x0000 (SNI) → server_name
  - 0x0033 (key_share) → 查找 X25519 (group=0x001D)
  - 0x002B (supported_versions) → versions[]
  - 0x0010 (ALPN) → has_alpn
  - 0xFE0D (ECH) → has_ech
```

## 设计决策

### 为什么用独立的 client_hello 类而非直接返回 hello_features？

**问题**: `hello_features` 是平铺结构体，所有字段都是 public，解析逻辑与数据结构混合。

**选择**: `client_hello` 封装解析过程，提供只读访问接口。解析完成后可按需转换为 `hello_features`。

**后果**: 新代码路径使用 `client_hello`，旧代码路径通过 `to_features()` 兼容。两种表示共存直到旧代码全部迁移。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `recognition` → `client_hello` | 调用 | recognition 解析 ClientHello 后提取特征 |
| `stealth` ← `client_hello` | 依赖 | 各方案的 sniff/verify 使用解析结果 |
| `client_hello` → [[core/protocol/tls/types\|types]] | 依赖 | 使用 `hello_features` 结构体 |
| `client_hello` → [[core/protocol/tls/record\|record]] | 依赖 | `from()` 接收 `tls::record` 输入 |

## 相关文档

- [[core/protocol/tls/types|TLS 类型定义]] — hello_features 和常量
- [[core/protocol/tls/record|TLS 记录帧]] — record 读写
- [[core/recognition/overview|Recognition]] — 协议识别流水线
- [[core/stealth/overview|Stealth]] — 伪装方案检测
