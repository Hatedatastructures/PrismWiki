---
title: TLS 记录帧
layer: core
source:
  - include/prism/protocol/tls/record.hpp
  - src/prism/protocol/tls/record.cpp
tags: [protocol, tls, record, framing]
---

# TLS 记录帧

> 源码: `include/prism/protocol/tls/record.hpp` | 实现: `src/prism/protocol/tls/record.cpp`

TLS 记录帧的读写和序列化抽象（RFC 8446 §5.1），统一替代各方案中的重复实现。支持 `transport::transmission` 和 `tcp::socket` 两种底层 I/O。

## 记录帧格式

```
TLS Record (RFC 8446 §5.1):
+------------+----------+----------+------------------+
| ContentType | Version  | Length   | Fragment        |
| (1 byte)    | (2 bytes)| (2 bytes)| (Length bytes)  |
+------------+----------+----------+------------------+
```

| 字段 | 大小 | 说明 |
|------|------|------|
| ContentType | 1 字节 | 记录类型（0x14=CCS, 0x15=Alert, 0x16=Handshake, 0x17=AppData） |
| Version | 2 字节 | 协议版本（TLS 1.2 记录层统一用 0x0303） |
| Length | 2 字节 | 载荷长度（最大 16384） |
| Fragment | Length 字节 | 加密或明文载荷 |

## 核心类

### `record_header`

5 字节记录头，存储 content_type、version、length。

### `record`

完整的 TLS 记录帧，包含 header 和 payload。

**协程 I/O**:

| 方法 | 底层 | 说明 |
|------|------|------|
| `read(transmission)` | `transport::transmission` | 从传输层读取记录 |
| `read(tcp::socket)` | `tcp::socket` | 从 socket 读取记录 |
| `write(transmission)` | `transport::transmission` | 向传输层写入记录 |
| `write(tcp::socket)` | `tcp::socket` | 向 socket 写入记录 |

**序列化**: `serialize()` 序列化为线路字节；`record::builder` 链式构建器。

## 设计决策

### 为什么统一 record 抽象？

**问题**: Reality seal、ShadowTLS、Restls 等方案各自实现了 TLS 记录读写逻辑，代码重复且行为不一致。

**选择**: 提取 `psm::tls::record` 作为统一抽象，所有方案共用。

**后果**: 单一实现减少了维护负担，但也意味着所有方案共享同一个记录帧格式解析逻辑。

### 为什么同时支持 transmission 和 tcp::socket？

**问题**: Reality 的 seal 传输层使用 `transport::transmission`，而底层场景（如 fetch_dest_cert）直接使用 `tcp::socket`。

**选择**: 提供两组重载，避免中间层包装。

**后果**: 两组 I/O 方法行为完全一致，仅底层调用不同。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `stealth/reality/seal` → `record` | 依赖 | seal 的 recv_record/send_record 使用 record::read/write |
| `stealth/reality/handshake` → `record` | 依赖 | 握手中读取客户端 Finished |
| `recognition/tls` → `record` | 依赖 | ClientHello 读取使用 record::read |
| `record` → [[core/transport/transmission\|transport]] | 依赖 | read/write 需要 transmission 接口 |

## 相关文档

- [[core/protocol/tls/types|TLS 类型定义]] — 常量（CT_*, RECORD_HDR_LEN 等）
- [[core/protocol/tls/hello|ClientHello 解析]] — 基于 record 解析 ClientHello
- [[core/stealth/reality/seal|Reality Seal]] — 使用 record 进行 TLS 1.3 记录读写
- [[core/recognition/overview|Recognition]] — 使用 record 读取 ClientHello
