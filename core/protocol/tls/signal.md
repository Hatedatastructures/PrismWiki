---
layer: core
source: "I:/code/Prism/include/prism/protocol/tls/signal.hpp"
---

# TLS ClientHello 解析器

> 源码位置: `I:/code/Prism/include/prism/protocol/tls/signal.hpp`

## 概述

TLS ClientHello 解析器，解析 TLS 记录层的 ClientHello 消息，提取 SNI、key_share 公钥、session_id 和 supported_versions 等关键字段。解析器是无状态的，所有方法均为纯函数。该模块是中立的共享层，供 recognition 和 stealth 模块共同使用。

## 命名空间

```cpp
namespace psm::protocol::tls
```

## 核心函数

### read_tls_record

读取完整的 TLS 记录。

```cpp
auto read_tls_record(channel::transport::transmission &transport)
    -> net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>;
```

**参数**:
- `transport`: 底层传输（应包含预读数据）

**返回**: 异步操作，返回错误码和完整 TLS 记录（含 5 字节 record header）

### read_tls_record (带预读)

读取完整的 TLS 记录，使用已预读的数据作为前缀。

```cpp
auto read_tls_record(channel::transport::transmission &transport, 
                     std::span<const std::byte> preread)
    -> net::awaitable<std::pair<fault::code, memory::vector<std::uint8_t>>>;
```

**参数**:
- `transport`: 底层传输
- `preread`: 已预读的数据

**返回**: 异步操作，返回错误码和完整 TLS 记录

### parse_client_hello

解析 ClientHello 并提取特征。

```cpp
[[nodiscard]] auto parse_client_hello(std::span<const std::uint8_t> record)
    -> std::pair<fault::code, client_hello_features>;
```

**参数**:
- `record`: 完整的 TLS 记录（含 record header）

**返回**: 错误码和解析后的特征结构

**提取字段**:
- SNI（Server Name Indication）
- key_share 公钥（X25519）
- session_id
- supported_versions

## 设计特性

**无状态性**:
- 所有方法均为纯函数
- 不维护内部状态
- 可安全并发调用

**中立共享层**:
- 不依赖任何具体实现
- 供 recognition 和 stealth 模块共同使用

## 调用链

- [[core/protocol/tls/types|Types]] - client_hello_features 结构定义
- [[core/protocol/tls/feature_bitmap|Feature Bitmap]] - 使用特征构建位图
- [[core/channel/transport/transmission|Transmission]] - 底层传输接口
- [[stealth/reality|Reality]] - Reality 协议检测
- [[stealth/ech|ECH]] - ECH 协议检测
- [[stealth/shadowtls|ShadowTLS]] - ShadowTLS 协议检测