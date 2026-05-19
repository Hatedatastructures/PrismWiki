---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/handshake.hpp
title: ShadowTLS v3 服务端握手
---

# ShadowTLS v3 服务端握手

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/handshake.hpp`

## 概述

ShadowTLS v3 服务端处理流程：
1. 接收已读取的 ClientHello（由 Recognition 层预读）
2. 验证 SessionID 中的 HMAC 标签
3. 认证成功后，与后端服务器完成 TLS 握手
4. 握手完成后，处理数据帧的 HMAC 验证
5. 创建 shadowtls_transport 传输层包装器，继承 HMAC 状态

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
    std::vector<std::byte> client_first_frame; // 客户端首帧数据（认证后，TLS header + payload）
    std::string_view matched_user;             // 匹配的用户名
    std::string matched_password;              // 匹配的密码（用于后续 HMAC 计算）
    std::array<std::byte, 32> server_random{}; // ServerHello 的 ServerRandom
    std::shared_ptr<HMAC_CTX> hmac_write_ctx;  // 写入方向累积 HMAC（初始：password + SR + "S")
    std::shared_ptr<HMAC_CTX> hmac_read_ctx;   // 读取方向累积 HMAC（初始：password + SR + "C" + payload + HMAC[:4])
};
```

**重要变更**：v3 传输阶段需要继承握手阶段的累积 HMAC 状态，否则客户端验证会失败。

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

**返回**：握手结果，包含认证状态、首帧数据和 HMAC 上下文

## HMAC 规则详解

参照 sing-shadowtls 源码，ShadowTLS v3 使用两阶段 HMAC：

### 握手阶段（Handshake Relay）

| 方向 | HMAC 计算 | 说明 |
|------|-----------|------|
| Client → Server (验证) | `HMAC-SHA1(password, SR + "C" + payload)[:4]` | 每帧独立验证 |
| Backend → Client (修改) | `HMAC-SHA1(password, SR || XOR'd_payloads)[:4]` | Relay 累积模式 |

握手阶段 backend → client 方向使用 XOR 加密（WriteKey = SHA256(password + SR)）。

### 传输阶段（Transport Layer）

| 方向 | HMAC 初始化 | HMAC 更新 | 说明 |
|------|-------------|-----------|------|
| Server → Client (写入) | `HMAC(password + SR + "S")` | `HMAC.Update(payload) → HMAC[:4] → HMAC.Update(HMAC[:4])` | 累积生成 |
| Client → Server (读取) | `HMAC(password + SR + "C" + first_payload + HMAC[:4])` | `HMAC.Update(payload) → verify → HMAC.Update(HMAC[:4])` | 累积验证 |

**关键点**：
- 传输阶段发送 **plain payload**，不 XOR 加密
- 每次写入/读取后，将 HMAC[:4] 也加入累积状态

## 数据帧处理流程

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
       │  ServerHello (原样转发)           │                                  │
       │<─────────────────────────────────│                                  │
       │                                  │                                  │
       │                                  │  Application Data (XOR + HMAC)   │
       │                                  │<─────────────────────────────────│
       │                                  │                                  │
       │  Application Data (修改后转发)    │                                  │
       │<─────────────────────────────────│                                  │
       │                                  │                                  │
       │  Application Data (HMAC 验证)     │                                  │
       │─────────────────────────────────>│                                  │
       │                                  │                                  │
       │                                  │  [握手完成，创建 transport]        │
       │                                  │                                  │
       │  Application Data (累积 HMAC)     │                                  │
       │<─────────────────────────────────>│                                  │
       │                                  │                                  │
```

## 调用链

```
stealth/scheme::handshake -> shadowtls::handshake::handshake
shadowtls::handshake::handshake -> shadowtls::auth::verify_client_hello
shadowtls::handshake::handshake -> read_until_hmac_match (初始化 hmac_read_ctx)
shadowtls::handshake::handshake -> relay_backend_to_client (握手阶段 HMAC)
shadowtls::handshake::handshake -> 创建 shadowtls_transport (继承 HMAC 状态)
```

## 依赖

- [[core/stealth/shadowtls/config|ShadowTLS 配置]] - 服务端配置
- [[core/stealth/shadowtls/auth|ShadowTLS 认证]] - HMAC 验证逻辑
- [[core/stealth/shadowtls/constants|ShadowTLS 常量]] - 协议常量
- [[core/stealth/shadowtls/transport|ShadowTLS Transport]] - 传输层包装器