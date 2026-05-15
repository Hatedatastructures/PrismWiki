---
title: "seal — Reality 加密传输层"
source: "include/prism/stealth/reality/seal.hpp"
module: "stealth"
submodule: "reality"
type: api
tags: [stealth, reality, seal, 加密, 传输层, aead]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/reality/handshake|handshake]]"
  - "[[stealth/reality/keygen|keygen]]"
  - "[[channel/transport/transmission|transmission]]"
  - "[[crypto/aead|aead]]"
  - "[[stealth/reality/constants|constants]]"
  - "[[agent/session/session|session]]"
---

# seal.hpp

> 源码: `include/prism/stealth/reality/seal.hpp`
> 实现: `src/prism/stealth/reality/seal.cpp`
> 模块: [[stealth|stealth]] > [[stealth/reality|reality]]

## 概述

Reality 加密传输层。实现 TLS 1.3 应用数据记录的加密/解密传输层，继承 [[channel/transport/transmission|transmission]] 接口，替代 BoringSSL 的 encrypted 传输层，提供 Reality 协议所需的 TLS 1.3 应用数据加密通道。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[channel/transport/transmission|transmission]] | 继承 transmission 接口 |
| 依赖 | [[crypto/aead|aead]] | 使用 AEAD 加解密（AES-128-GCM） |
| 依赖 | [[memory/container|container]] | 使用 memory::vector PMR 容器 |
| 依赖 | [[stealth/reality/constants|constants]] | 使用协议常量（AEAD_NONCE_LEN、AEAD_TAG_LEN） |
| 依赖 | [[stealth/reality/keygen|keygen]] | 使用 key_material 密钥材料 |
| 被依赖 | [[stealth/reality/handshake|handshake]] | 握手完成后创建加密传输层 |
| 被依赖 | [[agent/session/session|session]] | 会话中使用加密传输层读写数据 |

## 命名空间

`psm::stealth::reality`

---

## 类: seal

> 源码: `include/prism/stealth/reality/seal.hpp:36`

### 概述

Reality 加密传输层。封装 TLS 1.3 应用数据记录的加密/解密。读取时从底层传输读取加密的 TLS 记录后解密并缓冲明文，写入时将明文加密为 TLS 记录后写入底层传输。使用 AES-128-GCM AEAD 加密，nonce 由 IV 和序列号异或生成。

### 类层次

```
channel::transport::transmission [[channel/transport/transmission|transmission]]
  └── reality::seal
```

### 设计意图

Reality 需要自定义的加密传输层，因为：
- Reality 使用自定义的 X25519 共享密钥替代标准 TLS ECDHE 结果
- 需要精确控制 TLS 1.3 应用数据记录的格式
- 需要与标准 TLS 1.3 兼容，但使用不同的密钥来源

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| transport_ | shared_transmission | 底层传输连接 |
| keys_ | key_material | TLS 1.3 密钥材料 |
| server_encryptor_ | crypto::aead_context | 服务端加密上下文（用于写入） |
| client_decryptor_ | crypto::aead_context | 客户端解密上下文（用于读取） |
| read_sequence_ | uint64_t | 读取序列号，用于生成 nonce |
| write_sequence_ | uint64_t | 写入序列号，用于生成 nonce |
| plaintext_buffer_ | memory::vector\<std::byte\> | 解密后的明文缓冲区 |
| plaintext_offset_ | std::size_t | 明文缓冲区当前读取偏移 |

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| seal() | 构造加密传输层 | 下文 |
| is_reliable() | 始终返回 true | 下文 |
| executor() | 获取执行器 | 下文 |
| async_read_some() | 异步读取解密后的数据 | 下文 |
| async_write_some() | 异步加密写入数据 | 下文 |
| async_write_scatter() | Scatter-gather 加密写入 | 下文 |
| close() | 关闭传输层 | 下文 |
| cancel() | 取消所有未完成的异步操作 | 下文 |
| read_encrypted_record() | 读取并解密一个 TLS 记录 | 下文 |
| write_encrypted_record() | 加密并写入一个 TLS 记录 | 下文 |
| make_nonce() | 生成 AEAD nonce | 下文 |

### 生命周期

1. **构造**: 由 [[stealth/reality/handshake|handshake()]] 在认证成功后创建
2. **使用**: 作为加密传输层，被 [[agent/session/session|session]] 使用
3. **销毁**: 会话结束时自动销毁

### 线程安全

- 所有 I/O 操作需要在协程中调用
- 不支持并发读写

### 异常安全

- 所有函数不抛异常，错误通过 error_code 报告

---

## 函数: seal()

> 源码: `include/prism/stealth/reality/seal.hpp:46`

### 功能

构造加密传输层。使用密钥材料初始化 AES-128-GCM 加密和解密上下文，服务端密钥用于加密（写入），客户端密钥用于解密（读取）。

### 签名

```cpp
explicit seal(channel::transport::shared_transmission transport,
              key_material keys);
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| transport | shared_transmission | 底层传输连接 |
| keys | key_material | TLS 1.3 密钥材料（握手+应用阶段密钥） |

### 返回值

无（构造函数）

### 调用（向下）

- [[crypto/aead|aead_context]] — 初始化加密/解密上下文

### 被调用（向上）

- [[stealth/reality/handshake|handshake]] — 握手完成后创建 seal

### 知识域

- [[crypto/aead|aead]] — AEAD 加密上下文
- [[stealth/reality/keygen|key_material]] — TLS 1.3 密钥材料

---

## 函数: is_reliable()

> 源码: `include/prism/stealth/reality/seal.hpp:54`

### 功能

检查传输是否可靠。始终返回 `true`，seal 基于 TCP 传输。

### 签名

```cpp
[[nodiscard]] auto is_reliable() const noexcept -> bool override;
```

### 参数

无

### 返回值

`bool` — 始终返回 `true`

### 调用（向下）

- 无

### 被调用（向上）

- [[channel/transport/transmission|transmission]] — 传输层接口

### 知识域

- [[channel/transport/transmission|transmission]] — 传输层接口

---

## 函数: executor()

> 源码: `include/prism/stealth/reality/seal.hpp:61`

### 功能

获取执行器。返回底层传输的执行器，用于协程调度。

### 签名

```cpp
[[nodiscard]] auto executor() const -> executor_type override;
```

### 参数

无

### 返回值

`executor_type` — Boost.Asio 执行器

### 调用（向下）

- [[channel/transport/transmission|transmission::executor()]] — 底层传输的执行器

### 被调用（向上）

- Boost.Asio — 协程调度

### 知识域

- Boost.Asio — 执行器模型

---

## 函数: async_read_some()

> 源码: `include/prism/stealth/reality/seal.hpp:71`

### 功能

异步读取解密后的数据。优先从明文缓冲区 `plaintext_buffer_` 返回已解密数据；缓冲区耗尽后调用 `read_encrypted_record()` 从底层传输读取加密记录并解密填充缓冲区。

### 签名

```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buffer | std::span<std::byte> | 接收缓冲区 |
| ec | std::error_code& | 错误码输出参数 |

### 返回值

`net::awaitable<std::size_t>` — 异步操作，返回读取字节数

### 调用（向下）

- read_encrypted_record() — 读取并解密一个 TLS 记录
- [[crypto/aead|aead_context::open]] — AEAD 解密

### 被调用（向上）

- [[agent/session/session|session]] — 会话中读取数据
- [[stealth/reality/handshake|handshake]] — 握手完成后预读内层数据

### 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-128-GCM 认证加密
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 应用数据记录

---

## 函数: async_write_some()

> 源码: `include/prism/stealth/reality/seal.hpp:81`

### 功能

异步加密写入数据。将明文数据调用 `write_encrypted_record()` 加密为 TLS 记录后写入底层传输。

### 签名

```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buffer | std::span<const std::byte> | 发送缓冲区 |
| ec | std::error_code& | 错误码输出参数 |

### 返回值

`net::awaitable<std::size_t>` — 异步操作，返回写入字节数

### 调用（向下）

- write_encrypted_record() — 加密并写入一个 TLS 记录
- [[crypto/aead|aead_context::seal]] — AEAD 加密

### 被调用（向上）

- [[agent/session/session|session]] — 会话中写入数据

### 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-128-GCM 认证加密
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 应用数据记录

---

## 函数: async_write_scatter()

> 源码: `include/prism/stealth/reality/seal.hpp:93`

### 功能

Scatter-gather 加密写入。将多个缓冲区拼接到 `scatter_buf_` 后调用 `write_encrypted_record()` 一次性加密写入，避免多次加密和写入的系统调用开销。

### 签名

```cpp
auto async_write_scatter(const std::span<const std::byte> *buffers, std::size_t count,
                         std::error_code &ec) -> net::awaitable<std::size_t> override;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| buffers | const std::span<const std::byte>* | 缓冲区数组指针 |
| count | std::size_t | 缓冲区数量 |
| ec | std::error_code& | 错误码输出参数 |

### 返回值

`net::awaitable<std::size_t>` — 异步操作，返回写入字节数

### 调用（向下）

- write_encrypted_record() — 加密并写入一个 TLS 记录

### 被调用（向上）

- [[agent/session/session|session]] — 会话中 scatter-gather 写入

### 知识域

- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 应用数据记录

---

## 函数: close()

> 源码: `include/prism/stealth/reality/seal.hpp:100`

### 功能

关闭传输层。关闭底层传输连接，清空明文缓冲区。

### 签名

```cpp
void close() override;
```

### 参数

无

### 返回值

无

### 调用（向下）

- [[channel/transport/transmission|transmission::close()]] — 关闭底层传输

### 被调用（向上）

- [[agent/session/session|session]] — 会话结束时关闭

### 知识域

- [[channel/transport/transmission|transmission]] — 传输层接口

---

## 函数: cancel()

> 源码: `include/prism/stealth/reality/seal.hpp:106`

### 功能

取消所有未完成的异步操作。取消底层传输的挂起操作。

### 签名

```cpp
void cancel() override;
```

### 参数

无

### 返回值

无

### 调用（向下）

- [[channel/transport/transmission|transmission::cancel()]] — 取消底层传输

### 被调用（向上）

- [[agent/session/session|session]] — 会话取消时调用

### 知识域

- [[channel/transport/transmission|transmission]] — 传输层接口

---

## 函数: read_encrypted_record()

> 源码: `include/prism/stealth/reality/seal.hpp:116`

### 功能

从底层传输读取并解密一个 TLS 记录。读取 5 字节 TLS 记录头获取长度，再读取密文体，使用客户端密钥和 `read_sequence_` 生成 nonce 解密后将明文存入 `plaintext_buffer_`，重置 `plaintext_offset_`。递增 `read_sequence_`。

### 签名

```cpp
auto read_encrypted_record(std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| ec | std::error_code& | 错误码输出参数 |

### 返回值

`net::awaitable<std::size_t>` — 异步操作，返回解密后的明文长度

### 调用（向下）

- [[channel/transport/transmission|transmission::async_read_some]] — 读取加密数据
- make_nonce() — 生成 AEAD nonce
- [[crypto/aead|aead_context::open]] — AEAD 解密

### 被调用（向上）

- async_read_some() — 缓冲区耗尽时调用

### 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-128-GCM 解密
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 记录层格式

---

## 函数: write_encrypted_record()

> 源码: `include/prism/stealth/reality/seal.hpp:126`

### 功能

加密并写入一个 TLS 记录。使用服务端密钥和 `write_sequence_` 生成 nonce 加密数据，构造 TLS ApplicationData 记录后写入底层传输。递增 `write_sequence_`。

### 签名

```cpp
auto write_encrypted_record(std::span<const std::byte> data, std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| data | std::span<const std::byte> | 明文数据 |
| ec | std::error_code& | 错误码输出参数 |

### 返回值

`net::awaitable<std::size_t>` — 异步操作，返回写入字节数

### 调用（向下）

- make_nonce() — 生成 AEAD nonce
- [[crypto/aead|aead_context::seal]] — AEAD 加密
- [[channel/transport/transmission|transmission::async_write_some]] — 写入加密数据

### 被调用（向上）

- async_write_some() — 异步写入
- async_write_scatter() — scatter-gather 写入

### 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-128-GCM 加密
- [[ref/protocol/tls-1.3|TLS 1.3]] — TLS 1.3 记录层格式

---

## 函数: make_nonce()

> 源码: `include/prism/stealth/reality/seal.hpp:136`

### 功能

生成 AEAD nonce。将 IV 和序列号按字节异或生成 nonce。序列号以大端序编码，与 IV 的后 8 字节异或。

### 签名

```cpp
[[nodiscard]] auto make_nonce(std::span<const std::uint8_t> iv, std::uint64_t sequence) const
    -> std::array<std::uint8_t, tls::AEAD_NONCE_LEN>;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| iv | std::span<const std::uint8_t> | 初始化向量（12 字节） |
| sequence | std::uint64_t | 序列号 |

### 返回值

`std::array<std::uint8_t, tls::AEAD_NONCE_LEN>` — 生成的 nonce（12 字节）

### 调用（向下）

- 无（纯函数）

### 被调用（向上）

- read_encrypted_record() — 解密时生成 nonce
- write_encrypted_record() — 加密时生成 nonce

### 知识域

- [[ref/crypto/aes-gcm|AES-GCM]] — AES-GCM nonce 生成（RFC 5288）
