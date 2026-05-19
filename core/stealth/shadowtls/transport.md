---
layer: core
source: I:/code/Prism/include/prism/stealth/shadowtls/transport.hpp
title: ShadowTLS v3 传输层包装器
---

# ShadowTLS v3 传输层包装器

> 源码位置: `I:/code/Prism/include/prism/stealth/shadowtls/transport.hpp`

## 概述

`shadowtls_transport` 是 ShadowTLS v3 传输阶段的传输层包装器，持续处理 ShadowTLS 协议帧：
- **读取**：验证累积 HMAC，剥离 TLS header 和 HMAC[:4] 后返回 payload
- **写入**：计算累积 HMAC，添加 HMAC[:4] 标签，包装成 TLS ApplicationData frame

继承握手阶段的累积 HMAC 状态，确保状态连续性。

## 命名空间

```cpp
namespace psm::stealth::shadowtls
```

## 类定义

```cpp
class shadowtls_transport final : public channel::transport::transmission
{
public:
    explicit shadowtls_transport(
        net::ip::tcp::socket socket,
        std::string_view password,
        std::span<const std::byte> server_random,
        std::span<const std::byte> initial_data,
        std::shared_ptr<HMAC_CTX> hmac_write_ctx,
        std::shared_ptr<HMAC_CTX> hmac_read_ctx);

    ~shadowtls_transport() override;

    [[nodiscard]] bool is_reliable() const noexcept override { return true; }
    [[nodiscard]] executor_type executor() const override;

    auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;

    auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;

    auto async_write(std::span<const std::byte> data, std::error_code &ec)
        -> net::awaitable<std::size_t> override;

    auto async_write_scatter(const std::span<const std::byte> *buffers, 
                             std::size_t count, 
                             std::error_code &ec)
        -> net::awaitable<std::size_t> override;

    void close() override;
    void shutdown_write() override;
    void cancel() override;
};
```

## 构造参数

| 参数 | 说明 |
|------|------|
| `socket` | 原始 TCP socket（所有权转移） |
| `password` | ShadowTLS 密码 |
| `server_random` | ServerHello 中的 ServerRandom（32 字节） |
| `initial_data` | 握手期间已读取的第一帧 payload |
| `hmac_write_ctx` | 写入方向累积 HMAC（初始：password + SR + "S") |
| `hmac_read_ctx` | 读取方向累积 HMAC（初始：password + SR + "C" + first_payload + HMAC[:4]) |

## 累积 HMAC 机制

### 读取方向（Client → Server）

参照 sing-shadowtls `verifyApplicationData`：

```
1. HMAC.Update(payload)
2. expected_hmac = HMAC.Sum(nil)[:4]
3. verify(client_hmac == expected_hmac)
4. HMAC.Update(client_hmac)  ← 将 HMAC[:4] 加入累积状态
```

**状态链**：
```
初始: HMAC(password + SR + "C" + first_payload + HMAC[:4])
每次: HMAC.Update(payload) → HMAC.Update(HMAC[:4])
```

### 写入方向（Server → Client）

参照 sing-shadowtls `verifiedConn.write`：

```
1. HMAC.Update(payload)
2. hmac_hash = HMAC.Sum(nil)[:4]
3. HMAC.Update(hmac_hash)   ← 将 HMAC[:4] 加入累积状态
4. send TLS frame: [Header(5)] [HMAC[:4]] [plain payload]
```

**状态链**：
```
初始: HMAC(password + SR + "S")
每次: HMAC.Update(payload) → HMAC.Update(HMAC[:4])
```

## TLS Frame 格式

| 字段 | 大小 | 说明 |
|------|------|------|
| TLS Header | 5 | `[0x17, 0x03, 0x03, len_high, len_low]` |
| HMAC Tag | 4 | HMAC-SHA1[:4] |
| Payload | N | **Plain payload**，不 XOR 加密 |

**关键点**：传输阶段发送 plain payload，XOR 加密仅用于握手阶段（backend → client relay）。

## Scatter-Gather 写入优化

`async_write_scatter` 将多个 buffer 合并成一个完整 payload，一次性封装成 TLS frame：

```cpp
auto async_write_scatter(const std::span<const std::byte> *buffers, 
                         std::size_t count, 
                         std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

**必要性**：ShadowTLS 协议要求每个 TLS frame 包含一个完整的 HMAC+payload 单元。如果使用默认实现（循环调用 async_write），会导致 SS2022 等协议的响应被分成多个 TLS frame，破坏协议语义。

## XOR 密钥

XOR 密钥仅在握手阶段使用（`compute_write_key`）：

```
WriteKey = SHA256(password + serverRandom)
```

传输阶段不使用 XOR 加密。

## 数据流

```
┌──────────────┐    ┌──────────────────────┐    ┌──────────────┐
│   上层协议    │    │  shadowtls_transport │    │   TCP Socket │
│ (SS2022等)   │    │                      │    │              │
└──────┬───────┘    └──────┬───────────────┘    └──────┬───────┘
       │                   │                           │
       │  plain data       │                           │
       │──────────────────>│                           │
       │                   │  HMAC.Update(data)        │
       │                   │  frame = TLS + HMAC + data│
       │                   │──────────────────────────>│
       │                   │                           │
       │                   │  TLS frame                │
       │                   │<──────────────────────────│
       │                   │  HMAC.Update(payload)     │
       │                   │  verify HMAC[:4]          │
       │                   │  HMAC.Update(HMAC[:4])    │
       │  plain payload    │                           │
       │<──────────────────│                           │
       │                   │                           │
```

## 调用链

```
scheme::handshake -> handshake::handshake
handshake::handshake -> 创建 shadowtls_transport
protocol::handler -> shadowtls_transport::async_read_some
                   -> shadowtls_transport::async_write
                   -> shadowtls_transport::async_write_scatter
```

## 依赖

- [[core/stealth/shadowtls/handshake|ShadowTLS Handshake]] - 握手阶段，初始化 HMAC 状态
- [[core/channel/transport/transmission|Transmission 接口]] - 传输层抽象
- [[core/memory/container|PMR 容器]] - 内存管理