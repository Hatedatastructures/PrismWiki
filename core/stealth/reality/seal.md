---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/seal.hpp
title: Reality Seal
---

# Reality Seal

Reality 加密传输层实现。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/seal.hpp`

## 类定义

```cpp
class seal final : public transport::transmission
```

继承 [[transmission]] 接口，封装 TLS 1.3 应用数据记录的加密/解密。

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

- [[transmission]] ← 继承接口
- [[keygen]] ← 密钥材料
- [[constants]] ← AEAD 参数
- [[crypto-aead]] ← AEAD 加密
- [[handshake]] ← 创建 seal