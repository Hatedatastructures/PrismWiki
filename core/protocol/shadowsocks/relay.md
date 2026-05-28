---
layer: core
source: "I:/code/Prism/include/prism/protocol/shadowsocks/relay.hpp"
title: SS2022 协议中继器
tags: [protocol, shadowsocks, relay, aead, conn, handshake, sip022]
---

# SS2022 协议中继器

> 源码位置: `I:/code/Prism/include/prism/protocol/shadowsocks/relay.hpp`

## 概述

SS2022 (SIP022) 协议中继器，是一个 AEAD 加密传输层装饰器。与 Trojan/VLESS 不同，SS2022 relay 在整个会话生命周期内保持活跃，因为所有数据都经过 AEAD 加解密。handshake() 解密请求头、验证时间戳、解析地址后，relay 继续作为 transmission 提供加解密的读写操作。

## 类定义

```cpp
class relay : public transport::transmission, 
              public std::enable_shared_from_this<relay>
```

**继承关系**:
- `transport::transmission` - 统一传输层接口
- `std::enable_shared_from_this` - 安全共享指针管理

## 构造函数

```cpp
explicit relay(shared_transmission next_layer, const config &cfg,
               std::shared_ptr<salt_pool> salts);
```

**参数**:
- `next_layer`: 底层传输层，必须已建立连接
- `cfg`: SS2022 协议配置
- `salts`: Salt 重放保护池，worker 线程独占

## 核心方法

### executor

获取关联的执行器。

```cpp
auto executor() const -> executor_type override;
```

### async_read_some

从底层传输层读取 AEAD 加密帧并解密返回明文。

```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

**读取状态机**: header -> 解密 2B 长度 -> payload -> 解密 payload -> 返回数据

### async_write_some

将明文数据加密为 AEAD 帧后写入底层传输层。

```cpp
auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

**写入流程**: 将数据分块 -> 加密长度 + payload -> scatter-gather 写入底层

### close / cancel

关闭和取消操作。

```cpp
void close() override;
void cancel() override;
```

### handshake

执行 SS2022 握手。

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
```

**流程**:
- 读取请求 salt
- 派生会话密钥
- 解密固定/变长头
- 验证时间戳和 salt 唯一性
- 解析目标地址

**重要**: 握手成功后需调用 acknowledge() 发送响应

### acknowledge

发送 SS2022 握手响应。

```cpp
auto acknowledge() -> net::awaitable<fault::code>;
```

**设计**: 响应发送延迟到上游拨号成功后，避免拨号失败时客户端收到误导性的成功响应

### target

获取解析后的目标地址。

```cpp
[[nodiscard]] auto target() const noexcept -> const analysis::target &
```

## 内部成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `next_layer_` | `shared_transmission` | 底层传输层 |
| `config_` | `config` | SS2022 协议配置 |
| `salt_pool_` | `std::shared_ptr<salt_pool>` | Salt 重放保护池 |
| `decrypt_ctx_` | `std::unique_ptr<crypto::aead_context>` | 解密上下文 |
| `encrypt_ctx_` | `std::unique_ptr<crypto::aead_context>` | 加密上下文 |
| `method_` | `cipher_method` | 加密方法 |
| `key_salt_length_` | `std::size_t` | 密钥/salt 长度 |
| `psk_` | `std::vector<std::uint8_t>` | 解码后的 PSK |

## 工厂函数

```cpp
using shared_relay = std::shared_ptr<relay>;

inline auto make_relay(shared_transmission next_layer, const config &cfg,
                       std::shared_ptr<salt_pool> salts) -> shared_relay;
```

## 调用链

- [[core/protocol/shadowsocks/format|Format]] - 使用解析函数处理协议头部
- [[core/protocol/shadowsocks/constants|Constants]] - 加密方法和常量
- [[core/protocol/shadowsocks/config|Config]] - 配置结构
- [[core/protocol/shadowsocks/salts|Salts]] - Salt 重放保护
- [[core/crypto/aead|AEAD]] - AEAD 加密上下文
- [[core/transport/transmission|Transmission]] - 底层传输层接口
- [[core/protocol/analysis|Analysis]] - 目标地址封装

## 实现边界

- **timestamp_window 默认 30s**：两端时钟偏移超过 30 秒会导致 `timestamp_expired` 错误。过窄（如 1s）在时钟漂移时大量失败，过宽（如 3600s）降低重放保护
- **PSK 半初始化风险**：构造函数中 PSK 解码失败不抛异常，`decrypt_ctx_` 为空 unique_ptr。正常路径（handshake 先检查 `psk_.empty()`）不会触发
- **salt_pool 线性增长**：`unordered_map<string, time_point>` 每秒清理一次过期条目，高连接速率（1000/s）下约 60000 条目，无上限保护
- **"乐观响应"设计**：acknowledge() 延迟到上游拨号成功后发送，但握手成功后拨号失败时客户端看到空连接
- **UDP session_tracker 资源耗尽向量**：`get_or_create` 在 AEAD 解密前调用，无效 session 不会被清理
- **nonce 递增不同步**：错误信息只有 `crypto_error`，无法区分密钥错/nonce 错/数据损坏

详见 [[dev/debugging/deep-dive/protocol-boundaries|代理协议实现边界与认证深层分析]]