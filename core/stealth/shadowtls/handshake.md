---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/handshake.hpp
---

# ShadowTLS v3 服务端握手

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/handshake.hpp`

## 概述

ShadowTLS v3 服务端处理流程：
1. 接收已读取的 ClientHello（由 Recognition 层预读）
2. 验证 SessionID 中的 HMAC 标签
3. 认证成功后，与后端服务器完成 TLS 握手
4. 握手完成后，处理数据帧的 HMAC 验证和 XOR 解密

与 Reality 不同，ShadowTLS 使用标准 TLS 外层，认证发生在 ClientHello 阶段，不需要伪造证书。

## 命名空间

```cpp
namespace psm::stealth::shadowtls
```

## 结构定义

### handshake_result

ShadowTLS 握手结果。

```cpp
struct handshake_result
{
    bool authenticated{false};                 // 是否认证成功
    std::error_code error;                     // 错误码
    std::vector<std::byte> client_first_frame; // 客户端首帧数据（认证后）
    std::string_view matched_user;             // 匹配的用户名
};
```

## 核心函数

### handshake

执行完整的 ShadowTLS v3 握手流程。

```cpp
auto handshake(net::ip::tcp::socket &client_sock, 
               const config &cfg, 
               memory::vector<std::byte> client_hello)
    -> net::awaitable<handshake_result>;
```

**参数**：
- `client_sock`: 客户端 TCP socket
- `cfg`: ShadowTLS 配置
- `client_hello`: 已读取的完整 ClientHello 帧（含 TLS header）

**返回**：握手结果，包含认证状态和错误信息

## 握手流程

```
┌─────────────┐                    ┌─────────────┐                    ┌─────────────┐
│   Client    │                    │   Server    │                    │   Backend   │
└──────┬──────┘                    └──────┬──────┘                    └──────┬──────┘
       │                                  │                                  │
       │  ClientHello (SessionID+HMAC)    │                                  │
       │─────────────────────────────────>│                                  │
       │                                  │                                  │
       │                                  │  验证 HMAC 标签                   │
       │                                  │─────────────────────────────────>│
       │                                  │                                  │
       │                                  │  ClientHello (转发)              │
       │                                  │─────────────────────────────────>│
       │                                  │                                  │
       │                                  │  ServerHello + ... + Finished   │
       │                                  │<─────────────────────────────────│
       │                                  │                                  │
       │  ServerHello + ... + Finished    │                                  │
       │<─────────────────────────────────│                                  │
       │                                  │                                  │
       │  Application Data (HMAC+XOR)     │                                  │
       │<─────────────────────────────────>│                                  │
       │                                  │                                  │
```

## 数据帧处理

握手完成后的数据帧处理：

| 方向 | HMAC 计算 | 说明 |
|------|-----------|------|
| Client → Server | `HMAC-SHA1(password, serverRandom + "C" + payload)[:4]` | 验证客户端帧 |
| Server → Client | `HMAC-SHA1(password, serverRandom + "S" + payload)[:4]` | 生成服务端帧 |
| Backend → Client | `HMAC-SHA1(password, serverRandom + payload)[:4]` | Relay 累积模式 |

XOR 加密密钥：
```
WriteKey = SHA256(password + serverRandom)
```

## 调用链

```
stealth/scheme::handshake -> shadowtls::handshake::handshake
shadowtls::handshake::handshake -> shadowtls::auth::verify_client_hello
shadowtls::handshake::handshake -> shadowtls::auth::verify_frame_hmac
shadowtls::handshake::handshake -> shadowtls::auth::compute_write_hmac
```

## 依赖

- [[core/stealth/shadowtls/config|ShadowTLS 配置]] - 服务端配置
- [[core/stealth/shadowtls/auth|ShadowTLS 认证]] - HMAC 验证逻辑
- [[core/stealth/shadowtls/constants|ShadowTLS 常量]] - 协议常量