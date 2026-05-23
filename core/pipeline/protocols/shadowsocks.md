---
title: "shadowsocks.hpp — SS2022 协议处理管道"
layer: core
source: "I:/code/Prism/include/prism/pipeline/protocols/shadowsocks.hpp"
created: 2026-05-17
updated: 2026-05-17
---

# shadowsocks.hpp

> 源码: `include/prism/pipeline/protocols/shadowsocks.hpp`
> 实现: `src/prism/pipeline/protocols/shadowsocks.cpp`
> 模块: Pipeline / protocols

## 概述

SS2022 协议处理管道，处理 Shadowsocks 2022 协议流量。负责无正特征的 SS2022 协议检测、密钥验证和数据转发。

## 命名空间

`psm::pipeline`

---

## 函数: shadowsocks()

> 源码位置: `shadowsocks.hpp:29`
> 实现位置: `shadowsocks.cpp`

### 功能

SS2022 协议处理函数，包括 AEAD 解密验证、地址解析和双向隧道转发。

### 签名

```cpp
auto shadowsocks(session_context &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ctx | `session_context &` | 会话上下文 |
| data | `std::span<const std::byte>` | 预读数据 |

### 返回值

`net::awaitable<void>` — 异步操作对象

### 处理流程

```
+------------------+
| 包装入站传输      |
| wrap_with_preview() |
| use_global_mr=true |
+------------------+
         │
         ▼
+------------------+
| 获取 salt pool    |
| thread_local 实例|
| 无锁防重放        |
+------------------+
         │
         ▼
+------------------+
| 创建 SS2022 relay |
| shadowsocks::make_relay() |
+------------------+
         │
         ▼
+------------------+
| 执行握手          |
| relay::handshake() |
| AEAD 解密验证     |
| 时间戳验证        |
| 地址解析          |
+------------------+
         │
         ▼
+------------------+
| 乐观响应          |
| relay::acknowledge() |
| 先发送 ack 再拨号 |
+------------------+
         │
         ▼
+------------------+
| forward()         |
| relay 作为 inbound |
| AEAD 加解密持续   |
+------------------+
```

### 调用链（向下）

- [[core/pipeline/primitives#wrap_with_preview|wrap_with_preview]] — 包装入站传输（use_global_mr=true）
- `protocol::shadowsocks::make_relay` — 创建 SS2022 中继器
- `protocol::shadowsocks::relay::handshake` — 执行握手
- `protocol::shadowsocks::relay::acknowledge` — 发送确认
- [[core/pipeline/primitives#forward|forward]] — TCP 转发

### 被调用（向上）

- [[core/instance/dispatch/table|dispatch]] — 协议分发表注册为 SS2022 处理器

---

## SS2022 协议特点

### 无正特征

SS2022 是无正特征协议，通过排除法 fallback 检测：
- 没有 HTTP/SOCKS5/TLS 的特征
- 24 字节预读数据不符合已知协议特征
- 尝试 AEAD 解密验证

### AEAD 加密

SS2022 使用 AEAD 认证加密：
- AES-128-GCM-SIV
- AES-256-GCM-SIV
- ChaCha20-Poly1305-SIV

### 密钥派生

使用 BLAKE3 进行密钥派生：
```cpp
// 从 PSK 派生会话密钥
auto session_key = blake3::derive_key(psk, salt);
```

### salt pool 防重放

```cpp
// thread_local salt pool
thread_local auto &pool = get_salt_pool();

// 检查 salt 是否已使用
if (pool.contains(salt)) {
    return fault::code::replay_attack;
}

// 记录 salt
pool.insert(salt, timestamp);
```

### 时间戳验证

SS2022 包含时间戳，用于防止重放攻击：
```cpp
auto current_time = std::chrono::system_clock::now();
auto packet_time = decode_timestamp(payload);

if (abs(current_time - packet_time) > tolerance) {
    return fault::code::timestamp_error;
}
```

---

## 实现细节

### 乐观响应模式

SS2022 使用乐观响应，先发送 acknowledge 再拨号上游，减少 RTT：

```cpp
// AEAD 解密成功后立即发送 acknowledge
co_await relay->acknowledge();

// 然后拨号上游
co_await primitives::forward(ctx, SS2022Str, target, relay->release());
```

### AEAD 解密验证

```cpp
// 从预读数据中提取 salt 和 encrypted payload
auto salt = extract_salt(data);
auto encrypted = extract_payload(data);

// 派生会话密钥
auto key = derive_session_key(psk, salt);

// AEAD 解密
auto [ec, decrypted] = co_await aead_decrypt(key, encrypted);

if (fault::failed(ec)) {
    trace::debug("{} decrypt failed", SS2022Str);
    co_return;
}
```

### 地址解析

```cpp
// 解析目标地址
auto target = analysis::resolve(decrypted);
```

### relay 作为传输层

SS2022 的 relay 本身是传输层，负责持续的 AEAD 加解密：

```cpp
// relay 继承 transmission
class relay : public transport::transmission
{
    // async_read_some: 解密数据
    // async_write_some: 加密数据
};
```

### 注意事项

- **无正特征**: 通过排除法检测，需要配置 PSK
- **全局内存池**: 使用 `use_global_mr=true`
- **salt pool**: thread_local 防重放
- **乐观响应**: 减少 RTT，与 Mihomo 兼容

---

## 相关文档

- [[core/pipeline/overview|Pipeline 层总览]]
- [[core/pipeline/primitives|管道原语]]
- [[core/protocol/shadowsocks/config|SS2022 配置]]
- [[core/crypto/blake3|BLAKE3 密钥派生]]
- [[core/crypto/aead|AEAD 加密]]