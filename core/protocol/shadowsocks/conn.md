---
title: SS2022 AEAD 流加密中继器
layer: core
source:
  - include/prism/protocol/shadowsocks/conn.hpp
  - src/prism/protocol/shadowsocks/conn.cpp
tags: [protocol, shadowsocks, conn]
---

# SS2022 AEAD 流加密中继器

SS2022 (SIP022) 协议的核心传输装饰器，在底层传输层之上提供 AEAD 加解密的读写操作。与 Trojan/VLESS 不同，SS2022 的 `conn` 在整个会话生命周期内保持活跃——所有上行/下行数据都经过 AEAD 分帧加密和解密。

## 模块概述

`conn` 同时继承 `transport::transmission` 和 `std::enable_shared_from_this<conn>`，因此它可以作为传输层被 tunnel 消费，同时支持安全的共享指针管理。

**生命周期**：

```
make_conn 创建 → handshake（解密请求头、验证时间戳、解析地址）
  → acknowledge（发送服务端响应）→ 作为 inbound 传输层进入 tunnel 双向转发
```

握手完成后，`conn` 的 `async_read_some` / `async_write_some` 自动处理 AEAD 分帧，上层代码无需感知加密细节。

## 核心函数

### handshake()

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
```

执行 SS2022 握手的完整流程：

1. 读取客户端 salt（长度等于 `key_salt_len_`，16 或 32 字节）
2. Salt 重放检测（`salt_pool_->check_and_insert`）
3. 以 PSK + client_salt 为密钥材料，通过 BLAKE3 KDF 派生解密上下文
4. 读取并解密固定头（27 字节密文 → 11 字节明文），验证请求类型和时间戳窗口
5. 读取并解密变长头，解析目标地址（SOCKS5 风格 ATYP+ADDR+PORT）和初始 payload
6. 返回解析后的 `request` 结构和目标地址

**超时保护**：30 秒 deadline timer，超时自动 `cancel()` 底层传输层。

### acknowledge()

```cpp
auto acknowledge() -> net::awaitable<fault::code>;
```

发送 SS2022 服务端响应。响应被延迟到上游拨号成功后发送，避免拨号失败时客户端收到误导性的成功响应。

响应结构：`server_salt + AEAD(type=0x01, server_ts, client_salt, paddingLen=0) + AEAD(empty_payload)`

### async_read_some() / async_write_some()

实现 `transport::transmission` 接口。读取状态机：

```
优先返回 init_payload_（握手阶段携带的初始数据）
  → 从 decrypted_ 缓冲区返回剩余数据
  → fetch_chunk: 读取加密长度块(18B) → 解密 2B 长度
  → 读取加密 payload 块 → 解密 payload → 返回明文
```

写入流程：将明文分块（最大 `max_chunk_size = 0x3FFF`）→ 加密长度 + 加密 payload → scatter-gather 合并写入底层。

### derive_aead_context()

```cpp
auto derive_aead_context(std::span<const std::uint8_t> salt) const
    -> std::unique_ptr<crypto::aead_context>;
```

拼接 PSK + salt 作为密钥材料，使用 BLAKE3 KDF（context 为 SIP022 规范字符串 `"shadowsocks 2022 session subkey"`）派生会话密钥，构造 AEAD 上下文。

### send_response()

构建并写入完整的 SS2022 服务端响应，包含：随机生成的 server_salt、AEAD 加密的固定头（类型 0x01、服务端时间戳、客户端 salt 回显、paddingLen=0）、AEAD 加密的空初始 payload 块。三个部分合并为一次 `async_write` 调用。

## 设计决策

### WHY: 传输装饰器而非一次性握手器

- **Problem**：Trojan/VLESS 的协议层在握手后即释放传输层，后续数据为明文传输。SS2022 的所有数据都需 AEAD 加解密。
- **Choice**：`conn` 继承 `transport::transmission`，在 `async_read_some`/`async_write_some` 中持续进行 AEAD 分帧。握手完成后 `conn` 本身作为 inbound 传输层传入 `connect::forward`。
- **Consequence**：tunnel 层完全无感知加密；但 `conn` 的生命周期与整个会话绑定，内存占用高于一次性握手器。

### WHY: 延迟响应（acknowledge 分离）

- **Problem**：如果在握手阶段立即发送响应，上游拨号失败时客户端会认为连接已建立。
- **Choice**：将 `handshake()` 和 `acknowledge()` 分为两步。process 层先调用 `handshake()`，拨号成功后再调用 `acknowledge()`。
- **Consequence**：与 mihomo 行为一致（乐观响应）；但拨号失败时客户端看到的是连接重置而非明确的错误响应。

### WHY: thread_local salt_pool

- **Problem**：SIP022 规范要求精确匹配的 salt 重放检测，禁止 Bloom filter。多线程共享需要加锁。
- **Choice**：每个 worker 线程持有独立的 `thread_local salt_pool` 实例，无锁访问。
- **Consequence**：同一线程内 salt 唯一性得到保证；跨线程的 salt 重复不会被检测（但概率极低，因为 salt 为密码学随机值）。

## 约束

| 约束 | 值 | 来源 |
|------|----|------|
| PSK 长度 | 16 或 32 字节（Base64 解码后） | SIP022 规范 |
| 时间戳窗口 | 默认 30 秒，可配置 | `config_.timestamp_window` |
| 数据块最大 payload | 0x3FFF (16383) 字节 | SIP022 规范 |
| AEAD tag 长度 | 16 字节 | AES-GCM / ChaCha20-Poly1305 固定 |
| 加密长度块 | 2 + 16 = 18 字节 | SIP022 规范 |
| 固定头密文 | 11 + 16 = 27 字节 | type(1) + ts(8) + varHdrLen(2) + tag(16) |
| 握手超时 | 30 秒 | 硬编码 |

## 失败场景

### 1. PSK 配置错误

构造函数中 Base64 解码 PSK 失败时，`psk_` 为空，`decrypt_ctx_` 未初始化。`handshake()` 首先检查 `psk_.empty()`，立即返回 `fault::code::invalid_psk`，不会触发空指针解引用。

### 2. Salt 重放攻击

攻击者重放合法握手的 salt。`salt_pool_->check_and_insert()` 检测到重复 salt 后返回 `false`，`handshake()` 返回 `fault::code::replay_detected`。

### 3. 时间戳窗口过期

客户端时钟偏移超过 `config_.timestamp_window`（默认 30 秒）。`read_fixed_hdr()` 计算服务端时间与客户端时间的绝对差值，超过窗口时返回 `fault::code::timestamp_expired`。

### 4. AEAD 解密失败

数据在传输中被篡改或密钥不匹配。固定头解密返回 `fault::code::auth_failed`；payload 解密失败设置 `std::errc::protocol_error`。nonce 递增不同步也会导致此类错误，但错误信息只显示 `crypto_error`，无法区分具体原因。

### 5. 握手超时

客户端在 30 秒内未发送完整握手数据。deadline timer 触发 `next_layer_->cancel()`，`handshake()` 将 `operation_canceled` 转换为 `fault::code::timeout`。

## 跨模块契约

| 上游模块 | 契约 |
|----------|------|
| [[core/protocol/shadowsocks/process]] | 调用 `make_conn`、`handshake`、`acknowledge`，然后将 conn 作为 inbound 传入 `connect::forward` |
| [[core/crypto/aead]] | 提供 `aead_context::seal/open`，nonce 自动递增 |
| [[core/protocol/shadowsocks/salts]] | 提供 `salt_pool` 的重放检测，`thread_local` 实例 |
| [[core/protocol/shadowsocks/constants]] | 定义所有协议常量和 `cipher_method` 枚举 |
| [[core/protocol/shadowsocks/config]] | 提供 `config` 结构（PSK、method、timestamp_window） |
| [[core/transport/transmission]] | 底层传输层接口，`conn` 装饰其上的 AEAD 层 |
| [[core/connect/tunnel/forward]] | 消费 conn 作为 inbound 传输层 |

## 变更敏感度

| 修改点 | 影响范围 |
|--------|----------|
| AEAD nonce 管理 | `fetch_chunk` 和 `send_chunk` 的加解密行为，必须与客户端 nonce 递增策略严格一致 |
| `max_chunk_size` | 写入分块逻辑，修改会导致与客户端不兼容 |
| KDF context 字符串 | 密钥派生结果完全不同，无法与现有客户端互通 |
| `fixed_hdr_plain` / `fixed_hdr_size` | 固定头读取逻辑，修改会破坏协议兼容性 |
| `salt_pool` TTL | 重放检测窗口，过短导致合法连接被拒绝，过长占用内存 |

## 相关文档

- [[core/protocol/shadowsocks/process]] — 入口函数，编排 handshake → acknowledge → forward
- [[core/protocol/shadowsocks/constants]] — 协议常量定义
- [[core/protocol/shadowsocks/config]] — 配置结构
- [[core/protocol/shadowsocks/salts]] — Salt 重放保护池
- [[core/protocol/shadowsocks/format]] — 地址解析、PSK 解码
- [[core/protocol/shadowsocks/replay]] — 重放攻击防御
- [[core/protocol/shadowsocks/relay]] — 旧版 relay 文档
- [[core/crypto/aead]] — AEAD 加密上下文
- [[core/transport/transmission]] — 传输层接口
