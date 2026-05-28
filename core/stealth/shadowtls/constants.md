---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/constants.hpp
title: ShadowTLS v3 协议常量
tags:
  - stealth
  - shadowtls
  - constants
---

# ShadowTLS v3 协议常量

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/constants.hpp`

## 设计决策（WHY）

### 为什么 `session_id_length_index` 恰好是 43

这个值来自 TLS ClientHello 的固定前缀结构：TLS Record Header (5) + Handshake Header (4) + Client Version (2) + Client Random (32) = 43 字节。第 44 字节（偏移 43）是 SessionID 长度字段。ShadowTLS 要求 SessionID 恰好 32 字节，HMAC 标签嵌入在最后 4 字节（偏移 72-75）。

### 为什么 `tls_hmac_header_size` 是 9 而非 5

数据帧 = TLS Record Header (5) + HMAC Tag (4) + Payload。ShadowTLS 在标准 TLS ApplicationData 帧的 payload 前插入了 4 字节 HMAC 标签。读取时需要跳过 9 字节（而非标准的 5 字节）才能到达 payload。

### 为什么 HMAC 标签只有 4 字节

ShadowTLS 参照 `sing-shadowtls` 实现，使用 HMAC-SHA1 输出的前 4 字节。4 字节 = 32 位，碰撞概率约 1/2^32（约 43 亿次）。对于认证场景，这个安全性足够——攻击者无法在合理时间内穷举出正确的 HMAC。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| SessionID 必须 32 字节 | `tls_session_id_size` | ShadowTLS 不支持其他长度 |
| HMAC 标签固定 4 字节 | `hmac_size` | 不与 SHA1 全输出比较 |
| 常量为编译期 `constexpr` | 性能 | 零运行时开销 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| 所有 ShadowTLS 子模块 → `constants` | 依赖 | 偏移量和大小定义 |
| `auth` → `constants` | 调用 | HMAC 位置和大小 |
| `handshake` → `constants` | 调用 | 帧格式和内容类型 |
| `transport` → `constants` | 调用 | 帧头大小 |

## 概述

定义 ShadowTLS v3 协议中使用的固定常数值。这些常量与 TLS 1.3 记录层格式和 ShadowTLS 认证机制相关。

## 命名空间

```cpp
namespace psm::stealth::shadowtls
```

## TLS 记录层常量

```cpp
constexpr std::size_t tls_header_size = 5;          // TLS 记录头长度
constexpr std::size_t tls_random_size = 32;         // TLS Random 长度
constexpr std::size_t tls_session_id_size = 32;     // ShadowTLS 要求的 SessionID 长度
constexpr std::size_t hmac_size = 4;                // HMAC 标签长度（4 字节）
```

## TLS 内容类型

```cpp
constexpr std::uint8_t content_type_handshake = 0x16;
constexpr std::uint8_t content_type_application_data = 0x17;
```

| 值 | 类型 | 说明 |
|----|------|------|
| 0x16 | Handshake | 握手消息 |
| 0x17 | Application Data | 应用数据 |

## TLS 握手类型

```cpp
constexpr std::uint8_t handshake_type_client_hello = 0x01;
constexpr std::uint8_t handshake_type_server_hello = 0x02;
```

| 值 | 类型 | 说明 |
|----|------|------|
| 0x01 | ClientHello | 客户端问候消息 |
| 0x02 | ServerHello | 服务端问候消息 |

## TLS 版本号

```cpp
constexpr std::uint16_t tls_version_1_3 = 0x0304;
```

ShadowTLS v3 严格要求 TLS 1.3 连接（在 strict_mode 下）。

## TLS 扩展类型

```cpp
constexpr std::uint16_t extension_supported_versions = 43;
```

用于检测 ClientHello 中是否支持 TLS 1.3。

## SessionID 中 HMAC 的位置

```cpp
constexpr std::size_t session_id_length_index = 43;
```

计算方式：TLS Header(5) + Handshake Header(4) + Version(2) + Random(32) = 43

SessionID 32 字节，HMAC 在最后 4 字节：

```
Offset 43: SessionID Length (1 byte, value = 32)
Offset 44-75: SessionID (32 bytes)
  - Offset 44-71: 随机数据 (28 bytes)
  - Offset 72-75: HMAC 标签 (4 bytes)
```

## 数据帧头大小

```cpp
constexpr std::size_t tls_hmac_header_size = tls_header_size + hmac_size; // 9
```

Data frame header = TLS Header (5) + HMAC (4) = 9 bytes

## 常量使用示例

### SessionID HMAC 提取

```cpp
// ClientHello 结构中的偏移计算
auto session_id_len = client_hello[session_id_length_index];  // 应为 32
auto hmac_offset = session_id_length_index + 1 + tls_session_id_size - hmac_size;
auto hmac = std::span{client_hello}.subspan(hmac_offset, hmac_size);
```

### 数据帧解析

```cpp
// 数据帧格式: TLS Header (5) + HMAC (4) + Payload
auto tls_header = frame.subspan(0, tls_header_size);
auto hmac_tag = frame.subspan(tls_header_size, hmac_size);
auto payload = frame.subspan(tls_hmac_header_size);
```

## 调用链

```
shadowtls::auth::verify_client_hello -> constants::tls_header_size, hmac_size
shadowtls::auth::verify_frame_hmac -> constants::tls_random_size
shadowtls::handshake::handshake -> constants::content_type_application_data
```

## 依赖

- [[core/protocol/tls/types|TLS 类型]] - TLS 协议常量