---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/auth.hpp
title: ShadowTLS v3 认证逻辑
---

# ShadowTLS v3 认证逻辑

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/auth.hpp`

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