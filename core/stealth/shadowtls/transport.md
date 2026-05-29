---
layer: core
source: include/prism/stealth/shadowtls/transport.hpp
title: ShadowTLS v3 传输层包装器
tags:
  - stealth
  - shadowtls
  - transport
  - HMAC
  - cumulative
---

# ShadowTLS v3 传输层包装器

> 源码位置: `include/prism/stealth/shadowtls/transport.hpp`

## 设计决策（WHY）

### 为什么使用 `shared_ptr<HMAC_CTX>` 而非值语义

OpenSSL 的 `HMAC_CTX` 是不透明类型，不支持直接拷贝（`HMAC_CTX` 拷贝语义已废弃）。使用 `shared_ptr` 共享所有权，确保握手阶段的 HMAC 状态可以安全传递给 transport，且 transport 析构时正确释放。

### 为什么 `async_write_scatter` 需要特殊实现

ShadowTLS 协议要求每个 TLS frame 包含一个完整的 HMAC+payload 单元。如果使用默认实现（循环调用 `async_write`），SS2022 等协议的响应会被分成多个 TLS frame，破坏 ShadowTLS 协议的帧语义。scatter 将多个 buffer 拼接后一次性封装成一个 TLS frame。

### 为什么传输阶段发送 plain payload 不加密

XOR 加密仅在握手阶段的 relay 中使用。传输阶段时 TLS 握手已完成，外层 TLS 连接提供加密保护。内层再加密一次是冗余的。ShadowTLS v3 的设计是"外层 TLS 加密 + 内层 HMAC 完整性"，不需要内层加密。

### 为什么 HMAC 每次更新后还要 `Update(HMAC[:4])`

这是累积 HMAC 的关键：每次计算完 HMAC 标签后，将标签本身也加入 HMAC 状态。这意味着下一帧的 HMAC 输入不仅包含当前帧的 payload，还隐式包含之前所有帧的 HMAC 标签。这种链式结构确保帧顺序完整性——删除或重排任何帧都会导致后续 HMAC 验证失败。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| HMAC 上下文来自握手阶段 | 累积状态 | 不能重新初始化 |
| 每帧必须包含 5 字节 TLS header + 4 字节 HMAC | 帧格式 | 偏移固定 |
| `async_write_scatter` 必须拼接后一次封装 | 协议语义 | 分帧会破坏协议 |
| TLS header 固定 `[0x17, 0x03, 0x03, len_hi, len_low]` | TLS 1.3 兼容 | 伪装为 ApplicationData 记录 |
| 写入方向 HMAC 和读取方向 HMAC 独立 | 密码学设计 | 各自累积，互不影响 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| 累积 HMAC 验证失败 | 客户端数据被篡改或帧丢失 | 返回错误，连接终止 |
| TLS 帧头读取失败 | 连接中断 | 返回 `io_error` |
| PMR 分配失败 | 内存耗尽 | 异常 |
| HMAC_CTX 指针为空 | 握手阶段未正确初始化 | 未定义行为 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `transport` → `channel::transport::transmission` | 继承 | 实现 `async_read_some`/`async_write_some`/`async_write_scatter` |
| `handshake` → `transport` | 创建 | 传递 HMAC 上下文、socket、server_random |
| `protocol` → `transport` | 使用 | 协议处理器通过 transport 读写数据 |
| `transport` → `memory` | 依赖 | PMR 容器管理帧缓冲区 |

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