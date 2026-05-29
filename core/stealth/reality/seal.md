---
layer: core
source: include/prism/stealth/reality/seal.hpp
title: Reality Seal
tags:
  - stealth
  - reality
  - seal
  - transport
  - AES-128-GCM
---

# Reality Seal

Reality 加密传输层实现。

## 源码位置

- 头文件：`include/prism/stealth/reality/seal.hpp`

## 设计决策（WHY）

### 为什么使用双序列号而非单序列号

TLS 1.3 规定读写方向各自维护独立的序列号，从 0 开始递增。服务端的 `write_sequence_` 用于加密发送给客户端的记录，`read_sequence_` 用于解密客户端发送的记录。如果共用一个序列号，当读写交错时 AEAD nonce 会出错。

### 为什么 `async_read_some` 有内部缓冲

TLS 记录的加密粒度是"记录"（一个完整 TLS frame），但上层协议的读取粒度是"字节流"（任意大小的 buffer）。一次 TLS 记录可能包含 2^14 字节数据，而上层可能只请求 64 字节。`plaintext_buffer_` 缓存解密后的多余数据，下次读取时直接返回。

### 为什么 `async_write_scatter` 需要拼接

Shadowsocks 等内层协议的响应可能被分成多个 buffer（header + payload）。如果每个 buffer 单独加密为一个 TLS 记录，会产生大量小记录（每个有 5 字节记录头 + 16 字节 AEAD tag 的开销）。`async_write_scatter` 将多个 buffer 拼接后一次加密，减少开销和系统调用次数。

### 为什么使用 AES-128-GCM 而非 ChaCha20-Poly1305

Reality 模拟的 TLS 密码套件是 `TLS_AES_128_GCM_SHA256`。这个选择是在 `keygen` 层决定的（密钥长度 16 字节）。Reality 必须与客户端 TLS 库期望的密码套件完全一致。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| 序列号不可重用 | TLS 1.3 AEAD 安全性 | nonce 重用 = 密钥流重用 = 安全性崩塌 |
| 明文缓冲区生命周期 | 协程挂起恢复 | 协程挂起后缓冲区迭代器可能失效 |
| `key_material` 必须在 seal 生命周期内有效 | shared_ptr 语义 | seal 拷贝 key_material，不依赖外部 |
| 记录长度最大 65535 字节 | TLS 记录头 2 字节 | 无额外检查 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| AEAD 解密失败 | 密钥不匹配或数据被篡改 | 返回错误，连接终止 |
| AEAD tag 验证失败 | 记录被中间人修改 | 返回错误，检测到篡改 |
| 序列号溢出 | 超过 2^64（实际不可能） | nonce 重复 |
| 底层 transport 读取失败 | 连接中断 | 返回 `io_error` |
| 内部缓冲区 PMR 分配失败 | 内存耗尽 | 异常（非 fault::code 路径） |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `seal` → `transport::transmission` | 继承 | 实现 `async_read_some`/`async_write_some`/`async_write_scatter` |
| `seal` → `crypto::aead` | 调用 | 每次读写使用 AEAD seal/open |
| `seal` → `keygen` | 依赖 | 构造时接收 `key_material` |
| `seal` → `stealth::common` | 间接 | nonce 生成逻辑与 `make_aead_nonce` 一致 |
| `handshake` → `seal` | 创建 | Stage 5 创建 seal 实例 |
| `protocol` → `seal` | 使用 | 认证成功后，协议处理器通过 seal 的传输层读写数据 |

## 类定义

```cpp
class seal final : public transport::transmission
```

继承 [[core/transport/transmission|transmission]] 接口，封装 TLS 1.3 应用数据记录的加密/解密。

## 构造函数

```cpp
explicit seal(transport::shared_transmission transport,
              key_material keys)
```

初始化加密传输层：

- 服务端密钥用于加密（写入）
- 客户端密钥用于解密（读取）

## 核心方法

### async_read_some

```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override
```

异步读取解密后的数据。

流程：
1. 优先从 plaintext_buffer_ 返回已解密数据
2. 缓冲区耗尽后读取加密记录
3. 解密后填充缓冲区

### async_write_some

```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override
```

异步加密写入数据。

流程：
1. 将明文数据加密为 TLS 记录
2. 写入底层传输

### async_write_scatter

```cpp
auto async_write_scatter(const std::span<const std::byte> *buffers, std::size_t count,
                         std::error_code &ec) -> net::awaitable<std::size_t> override
```

Scatter-gather 加密写入。

- 拼接多个缓冲区
- 一次性加密写入
- 减少系统调用开销

## 加密细节

### Nonce 生成

```cpp
[[nodiscard]] auto make_nonce(std::span<const std::uint8_t> iv, std::uint64_t sequence) const
    -> std::array<std::uint8_t, 12>
```

nonce = IV XOR sequence（字节级异或）

### 序列号

- `read_sequence_`：读取序列号
- `write_sequence_`：写入序列号

每个记录后序列号递增。

## 加密记录格式

```
TLS 1.3 Application Data Record:
[ContentType: 0x17][Version: 0x0303][Length: 2B][Ciphertext + Tag]

解密后:
[Plaintext][ContentType: 0x17]
```

## 内部缓冲区

| 缓冲区 | 用途 |
|--------|------|
| `plaintext_buffer_` | 解密后的明文缓冲区 |
| `record_body_buf_` | TLS 记录体读取缓冲区 |
| `decrypted_buf_` | 解密输出缓冲区 |
| `write_plain_buf_` | 写入明文拼接缓冲区 |
| `write_ciphertext_buf_` | 写入密文缓冲区 |
| `scatter_buf_` | scatter-gather 拼接缓冲区 |

## AEAD 加密

使用 AES-128-GCM：

- 密钥长度：16 字节
- IV 长度：12 字节
- Tag 长度：16 字节

## 调用链

- [[core/transport/transmission|transmission]] ← 继承接口
- [[keygen]] ← 密钥材料
- [[constants]] ← AEAD 参数
- [[core/crypto/aead|crypto-aead]] ← AEAD 加密
- [[handshake]] ← 创建 seal