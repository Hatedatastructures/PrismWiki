---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/auth.hpp
title: ShadowTLS v3 认证逻辑
tags:
  - stealth
  - shadowtls
  - auth
  - HMAC-SHA1
---

# ShadowTLS v3 认证逻辑

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/auth.hpp`

## 设计决策（WHY）

### 为什么 HMAC 输入跳过 HMAC 位置本身

`verify_client_hello` 的 HMAC 计算方式是 `HMAC-SHA1(password, ClientHello[10:hmac_index] + 00000000 + ClientHello[hmac_index+4:])`。将 HMAC 位置替换为 4 个零字节是因为：HMAC 标签本身无法参与自己的计算（循环依赖）。用零占位保持了 ClientHello 的字节长度不变。

### 为什么使用 HMAC-SHA1 而非 HMAC-SHA256

ShadowTLS 协议参照 `sing-shadowtls` 实现，后者选择 HMAC-SHA1 是为了兼容性——SHA1 在所有 TLS 库中都可用。虽然 SHA1 已不推荐用于数字签名，但在 HMAC 模式下（作为 PRF）的安全性仍然足够。

### 为什么 `compute_write_key` 使用 SHA256 而非 HKDF

WriteKey 用于 XOR 加密（仅握手阶段的 relay），不是 AEAD 密钥。简单的 `SHA256(password + serverRandom)` 提供足够的伪随机性。使用 HKDF 会增加不必要的复杂度。注意：传输阶段不使用 XOR 加密，只有握手阶段的 relay 使用。

### 为什么传输阶段使用累积 HMAC 而握手阶段是单次 HMAC

握手阶段的 `verify_client_hello` 只验证一次 ClientHello 中的 SessionID HMAC。传输阶段每帧都需要 HMAC 验证，且每帧的 HMAC 输入包含之前所有帧的数据（累积 HMAC）。累积 HMAC 确保帧顺序和完整性——删除或重排任何一帧会导致后续所有帧的 HMAC 验证失败。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| `verify_client_hello` 需要完整 ClientHello（含 TLS 记录头） | 偏移计算 | 缺少记录头导致偏移错位 |
| `server_random` 必须 32 字节 | TLS 规范 | SHA256 输入长度固定 |
| WriteKey 仅用于握手 relay | 协议设计 | 传输阶段使用累积 HMAC |
| 所有 HMAC 函数为同步 | 无 I/O | 纯计算，可在任何上下文调用 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| SessionID 长度不等于 32 | 非 ShadowTLS 客户端 | `verify_client_hello` 返回 false |
| HMAC 不匹配 | 密码错误 | 认证失败 |
| 累积 HMAC 状态丢失 | transport 跨帧 | 后续所有帧验证失败 |
| ServerRandom 读取错误 | 后端 TLS 握手异常 | 无法计算 HMAC |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `auth` → `constants` | 依赖 | 使用 `hmac_size`、`tls_header_size`、`session_id_length_index` |
| `scheme::verify()` → `auth` | 调用 | Tier 1 验证时调用 `verify_client_hello` |
| `handshake` → `auth` | 调用 | 握手阶段使用 `verify_frame_hmac`、`compute_write_hmac`、`compute_write_key` |
| `transport` → `auth` | 间接 | transport 的累积 HMAC 初始状态由 handshake 传递 |

## 概述

基于 HMAC-SHA1 的认证机制。ShadowTLS v3 在 TLS ClientHello 的 SessionID 字段中嵌入 4 字节 HMAC 标签进行身份验证。

## 命名空间

```cpp
namespace psm::stealth::shadowtls
```

## 认证算法

参照 `sing-shadowtls v3_server.go verifyClientHello`：

```
HMAC = HMAC-SHA1(password, ClientHello[10:hmac_index] + 00000000 + ClientHello[hmac_index+4:])[:4]
```

握手完成后的数据帧处理：

| 方向 | HMAC 计算 |
|------|-----------|
| Client → Server | `HMAC-SHA1(password, serverRandom + "C" + payload)[:4]` |
| Server → Client | `HMAC-SHA1(password, serverRandom + "S" + payload)[:4]` |
| Relay (Backend → Client) | `HMAC-SHA1(password, serverRandom + payload)[:4]`（累积，无后缀） |
| WriteKey | `SHA256(password + serverRandom)` |

## 核心函数

### verify_client_hello

验证 ClientHello 中的 SessionID HMAC。

```cpp
[[nodiscard]] auto verify_client_hello(std::span<const std::byte> client_hello,
                                        std::string_view password) -> bool;
```

**参数**：
- `client_hello`: ClientHello 帧数据（含 TLS 记录头）
- `password`: 认证密码

**返回**: `true` 如果认证通过

**细节**: 从 ClientHello 帧中提取 SessionID 的后 4 字节 HMAC 标签，与本地计算结果比较。

### compute_hmac

计算 HMAC-SHA1 标签。

```cpp
[[nodiscard]] auto compute_hmac(std::string_view key, 
                                const std::byte *data, 
                                std::size_t data_len)
    -> std::array<std::uint8_t, 4>;
```

**返回**: 4 字节 HMAC 标签

### verify_frame_hmac

验证握手后的数据帧 HMAC（客户端→服务端方向，单帧独立）。

```cpp
[[nodiscard]] auto verify_frame_hmac(std::string_view password,
                                      std::span<const std::byte> server_random,
                                      std::span<const std::byte> payload,
                                      std::span<const std::uint8_t, 4> client_hmac) -> bool;
```

**参数**：
- `password`: 密码
- `server_random`: TLS ServerRandom（32 字节）
- `payload`: 数据帧 payload（不含 TLS header 和 HMAC）
- `client_hmac`: 客户端帧头中的 4 字节 HMAC

**返回**: `true` 如果匹配

**注意**: 此函数用于单帧独立 HMAC。handshake.cpp 中使用累积 HMAC 模式。

### compute_write_hmac

生成数据帧的 HMAC 标签（服务端→客户端方向）。

```cpp
[[nodiscard]] auto compute_write_hmac(std::string_view password,
                                        std::span<const std::byte> server_random,
                                        std::span<const std::byte> payload)
    -> std::array<std::uint8_t, 4>;
```

**返回**: 4 字节 HMAC 标签

### compute_write_key

生成写入密钥（XOR 加密用）。

```cpp
[[nodiscard]] auto compute_write_key(std::string_view password,
                                      std::span<const std::byte> server_random)
    -> std::vector<std::uint8_t>;
```

**返回**: SHA256(password + serverRandom) 的输出

## HMAC 标签位置

```
ClientHello 结构:
┌─────────────────────────────────────────────────────────────┐
│ TLS Header (5 bytes)                                        │
├─────────────────────────────────────────────────────────────┤
│ Handshake Header (4 bytes)                                  │
├─────────────────────────────────────────────────────────────┤
│ Version (2) + Random (32) + SessionID Length (1)            │
├─────────────────────────────────────────────────────────────┤
│ SessionID (32 bytes) ──────────────────────────────────────┤
│                         │                                    │
│                         └──► Last 4 bytes: HMAC Tag ◄──────┤
├─────────────────────────────────────────────────────────────┤
│ ... (Cipher Suites, Extensions, etc.)                       │
└─────────────────────────────────────────────────────────────┘
```

## 调用链

```
shadowtls::handshake::handshake -> shadowtls::auth::verify_client_hello
shadowtls::handshake::handshake -> shadowtls::auth::verify_frame_hmac
shadowtls::handshake::handshake -> shadowtls::auth::compute_write_hmac
shadowtls::handshake::handshake -> shadowtls::auth::compute_write_key
```

## 依赖

- [[core/stealth/shadowtls/constants|ShadowTLS 常量]] - HMAC 长度、SessionID 长度等