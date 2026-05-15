---
title: "relay — SS2022 AEAD 流加密中继器"
source: "include/prism/protocol/shadowsocks/relay.hpp"
module: "protocol"
type: api
tags: [protocol, shadowsocks, relay, aead, 中继, 加密, BLAKE3]
related:
  - "[[protocol/shadowsocks/format|format]]"
  - "[[protocol/shadowsocks/constants|constants]]"
  - "[[protocol/shadowsocks/config|config]]"
  - "[[protocol/shadowsocks/salts|salts]]"
  - "[[channel/transport/transmission|transmission]]"
created: 2026-05-15
updated: 2026-05-15
---

# relay.hpp

> 源码: `include/prism/protocol/shadowsocks/relay.hpp`
> 实现: `src/prism/protocol/shadowsocks/relay.cpp`
> 模块: [[protocol|protocol]] > shadowsocks

## 概述

SS2022 AEAD 流加密中继器。继承 `transmission`，在底层传输层之上添加 SS2022 协议的 AEAD 加解密功能。与 Trojan/VLESS 不同，SS2022 relay 在整个会话生命周期内保持活跃，因为所有数据都经过 AEAD 加解密。`handshake()` 解密请求头、验证时间戳、解析地址后，relay 继续作为 `transmission` 提供加解密的读写操作。

核心特性：
- AEAD 加密：AES-128/256-GCM 或 ChaCha20-Poly1305
- BLAKE3 HKDF 密钥派生
- 分块传输：最大 0x3FFF 字节/块
- 长度加密：每个数据块的长度也加密
- Salt 重放保护

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 继承 | [[channel/transport/transmission|transmission]] | 传输层接口 |
| 依赖 | [[crypto/aead|aead]] | AEAD 加解密上下文 |
| 依赖 | [[protocol/shadowsocks/constants|constants]] | 协议常量 |
| 依赖 | [[protocol/shadowsocks/format|format]] | 格式编解码 |
| 依赖 | [[protocol/shadowsocks/config|config]] | 协议配置 |
| 依赖 | [[protocol/shadowsocks/salts|salts]] | Salt 重放保护池 |
| 依赖 | [[protocol/analysis|analysis]] | 目标地址结构 |
| 依赖 | [[memory|memory]] | PMR 容器 |
| 被依赖 | [[pipeline/protocols/shadowsocks|pipeline]] | 协议处理管道 |

## 命名空间

`psm::protocol::shadowsocks`

---

## 类: relay

> 源码: `include/prism/protocol/shadowsocks/relay.hpp:38`

### 概述

SS2022 AEAD 流加密中继器。`handshake()` 完成后，`async_read_some`/`async_write_some` 自动处理 AEAD 分帧加密/解密。

读取状态机：header -> 解密 2B 长度 -> payload -> 解密 payload -> 返回数据
写入流程：将数据分块 -> 加密长度+payload -> scatter-gather 写入底层

### 类层次

```
channel::transport::transmission
  +-- shadowsocks::relay (继承 transmission + enable_shared_from_this)
```

### 构造函数

```cpp
explicit relay(shared_transmission next_layer, const config &cfg,
               std::shared_ptr<salt_pool> salts);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `next_layer` | `shared_transmission` | 底层传输层，必须已建立连接 |
| `cfg` | `const config &` | SS2022 协议配置 |
| `salts` | `std::shared_ptr<salt_pool>` | Salt 重放保护池，worker 线程独占 |

---

### 函数: handshake()

#### 功能说明

执行 SS2022 握手。读取请求 salt，派生会话密钥，解密固定/变长头，验证时间戳和 salt 唯一性，解析目标地址。握手成功后需调用 `acknowledge()` 发送响应。

#### 签名

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
```

#### 参数

无

#### 返回值

`net::awaitable<std::pair<fault::code, request>>` — 错误码和请求信息。

#### 调用（向下）

- `next_layer_->async_read_some()` — 读取 salt 和加密数据
- `derive_aead_context()` — 派生 AEAD 上下文
- `read_fixed_header()` — 读取并验证加密固定头
- `read_variable_header()` — 读取并解析加密变长头
- [[protocol/shadowsocks/salts|salts]] `check_and_insert()` — Salt 重放检测
- [[protocol/shadowsocks/format|format]] `parse_address_port()` — 解析地址

#### 被调用（向上）

- [[pipeline/protocols/shadowsocks|pipeline]] 协议处理管道

#### 知识域

SS2022 握手流程、BLAKE3 HKDF、AEAD 解密

---

### 函数: acknowledge()

#### 功能说明

发送 SS2022 握手响应。必须在 `handshake()` 成功后调用。将响应发送延迟到上游拨号成功后，避免拨号失败时客户端收到误导性的成功响应。

#### 签名

```cpp
auto acknowledge() -> net::awaitable<fault::code>;
```

#### 参数

无

#### 返回值

`net::awaitable<fault::code>` — 错误码。

#### 调用（向下）

- `send_response()` — 构建并发送响应

#### 被调用（向上）

- [[pipeline/protocols/shadowsocks|pipeline]] 拨号成功后调用

#### 知识域

SS2022 响应格式

---

### 函数: target()

#### 功能说明

获取解析后的目标地址。

#### 签名

```cpp
[[nodiscard]] auto target() const noexcept -> const analysis::target &;
```

#### 参数

无

#### 返回值

`const analysis::target &` — 目标地址的常量引用。

#### 调用（向下）

无

#### 被调用（向上）

- [[pipeline/protocols/shadowsocks|pipeline]] 获取目标地址

---

### 函数: executor()

#### 功能说明

获取底层传输层的执行器。

#### 签名

```cpp
executor_type executor() const override;
```

#### 参数

无

#### 返回值

`executor_type` — 执行器。

#### 调用（向下）

- `next_layer_->executor()`

#### 被调用（向上）

- Boost.Asio 协程调度

#### 知识域

- [[channel/transport/transmission|transmission]] Boost.Asio 执行器

---

### 函数: async_read_some()

#### 功能说明

从底层传输层读取 AEAD 帧并解密。内部调用 `fetch_chunk()` 读取加密长度和 payload，解密后返回明文数据。

#### 签名

```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<std::byte>` | 接收缓冲区 |
| `ec` | `std::error_code &` | 错误码输出 |

#### 返回值

`net::awaitable<std::size_t>` — 读取的明文字节数。

#### 调用（向下）

- `fetch_chunk()` — 读取并解密 AEAD 数据块

#### 被调用（向上）

- [[pipeline/protocols/shadowsocks|pipeline]] 数据转发

#### 知识域

- [[crypto/aead|aead]] AEAD 解密
- [[protocol/shadowsocks/relay|relay]] SS2022 分块传输

---

### 函数: async_write_some()

#### 功能说明

将明文数据加密为 AEAD 帧后写入底层传输层。内部调用 `send_chunk()` 将数据分块加密后写入。

#### 签名

```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `std::span<const std::byte>` | 发送缓冲区（明文） |
| `ec` | `std::error_code &` | 错误码输出 |

#### 返回值

`net::awaitable<std::size_t>` — 写入的明文字节数。

#### 调用（向下）

- `send_chunk()` — 加密并写入 AEAD 数据块

#### 被调用（向上）

- [[pipeline/protocols/shadowsocks|pipeline]] 数据转发

#### 知识域

- [[crypto/aead|aead]] AEAD 加密
- [[protocol/shadowsocks/relay|relay]] SS2022 分块传输

---

### 函数: close()

#### 功能说明

关闭底层传输层连接。

#### 签名

```cpp
void close() override;
```

#### 参数

无

#### 返回值

无

#### 调用（向下）

- `next_layer_->close()`

#### 被调用（向上）

- session 析构

#### 知识域

- [[channel/transport/transmission|transmission]] 连接生命周期

---

### 函数: cancel()

#### 功能说明

取消所有未完成的异步操作。

#### 签名

```cpp
void cancel() override;
```

#### 参数

无

#### 返回值

无

#### 调用（向下）

- `next_layer_->cancel()`

#### 被调用（向上）

- session 超时处理

#### 知识域

- [[channel/transport/transmission|transmission]] 异步操作取消

---

### 函数: derive_aead_context()（私有）

#### 功能说明

从 PSK + salt 派生 AEAD 上下文。使用 BLAKE3 HKDF 派生子密钥。

#### 签名

```cpp
[[nodiscard]] auto derive_aead_context(std::span<const std::uint8_t> salt) const
    -> std::unique_ptr<crypto::aead_context>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `salt` | `std::span<const std::uint8_t>` | 盐值 |

#### 返回值

`std::unique_ptr<crypto::aead_context>` — AEAD 上下文智能指针。

#### 调用（向下）

- [[crypto/aead|aead]] AEAD 上下文创建
- BLAKE3 HKDF 密钥派生

#### 被调用（向上）

- `handshake()` 派生解密上下文

#### 知识域

BLAKE3 HKDF、AEAD 密钥派生

---

### 函数: read_fixed_header()（私有）

#### 功能说明

读取并验证加密固定头。解密 type(1) + timestamp(8) + varHeaderLen(2) = 11 字节明文。

#### 签名

```cpp
auto read_fixed_header() const
    -> net::awaitable<std::tuple<fault::code, std::uint16_t, std::int64_t>>;
```

#### 返回值

`tuple<fault::code, uint16_t, int64_t>` — 错误码、变长头长度、时间戳。

#### 调用（向下）

- `next_layer_->async_read_some()` — 读取加密数据
- [[crypto/aead|aead]] `decrypt()` — AEAD 解密

#### 被调用（向上）

- `handshake()` 握手流程

#### 知识域

SS2022 固定头格式

---

### 函数: read_variable_header()（私有）

#### 功能说明

读取并解析加密变长头。解析地址 + padding + 初始 payload。

#### 签名

```cpp
auto read_variable_header(std::uint16_t var_header_len, request &req)
    -> net::awaitable<fault::code>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `var_header_len` | `std::uint16_t` | 变长头长度 |
| `req` | `request &` | 请求结构，解析结果填充到此对象 |

#### 返回值

`net::awaitable<fault::code>` — 错误码。

#### 调用（向下）

- `next_layer_->async_read_some()` — 读取加密数据
- [[crypto/aead|aead]] `decrypt()` — AEAD 解密
- [[protocol/shadowsocks/format|format]] `parse_address_port()` — 解析地址

#### 被调用（向上）

- `handshake()` 握手流程

#### 知识域

SS2022 变长头格式

---

### 函数: fetch_chunk()（私有）

#### 功能说明

读取并解密下一个数据块到 `decrypted_` 缓冲区。读取 2 字节加密长度，解密获取 payload 大小，然后读取加密 payload 并解密。

#### 签名

```cpp
auto fetch_chunk(std::error_code &ec) -> net::awaitable<void>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `ec` | `std::error_code &` | 错误码输出 |

#### 返回值

`net::awaitable<void>`

#### 调用（向下）

- `next_layer_->async_read_some()` — 读取加密数据
- [[crypto/aead|aead]] `decrypt()` — AEAD 解密

#### 被调用（向上）

- `async_read_some()` 读取数据时

#### 知识域

SS2022 数据块格式

---

### 函数: send_chunk()（私有）

#### 功能说明

加密并写入一个数据块。将明文数据加密为 [长度块(18B) + payload块(数据+16B tag)] 格式。

#### 签名

```cpp
auto send_chunk(std::span<const std::byte> data, std::error_code &ec)
    -> net::awaitable<std::size_t>;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `data` | `std::span<const std::byte>` | 待加密的明文数据 |
| `ec` | `std::error_code &` | 错误码输出 |

#### 返回值

`net::awaitable<std::size_t>` — 实际写入的字节数。

#### 调用（向下）

- [[crypto/aead|aead]] `encrypt()` — AEAD 加密
- `next_layer_->async_write_some()` — 写入底层

#### 被调用（向上）

- `async_write_some()` 写入数据时

#### 知识域

SS2022 数据块格式

---

## 工厂函数: make_relay()

### 功能说明

工厂函数，创建 SS2022 AEAD 中继器实例。封装 `std::make_shared` 调用，返回 `shared_relay`。

### 签名

```cpp
inline auto make_relay(shared_transmission next_layer, const config &cfg,
                       std::shared_ptr<salt_pool> salts) -> shared_relay;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `next_layer` | `shared_transmission` | 底层传输层 |
| `cfg` | `const config &` | SS2022 协议配置 |
| `salts` | `std::shared_ptr<salt_pool>` | Salt 重放保护池 |

### 返回值

`shared_relay` — SS2022 中继器共享指针。

### 调用（向下）

- `std::make_shared<relay>(next_layer, cfg, salts)`

### 被调用（向上）

- [[pipeline/protocols/shadowsocks|pipeline]] 创建中继器

### 知识域

- [[protocol/shadowsocks/relay|relay]] SS2022 中继器
- [[pipeline/protocols/shadowsocks|pipeline]] 协议处理管道

## 相关页面

- [[protocol/shadowsocks/format|format]] — 格式编解码
- [[protocol/shadowsocks/constants|constants]] — 协议常量
- [[protocol/shadowsocks/config|config]] — 协议配置
- [[protocol/shadowsocks/salts|salts]] — Salt 重放保护池
- [[crypto/aead|aead]] — AEAD 加解密
