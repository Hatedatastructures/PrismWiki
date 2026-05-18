---
layer: core
source: "I:/code/Prism/include/prism/protocol/trojan/relay.hpp"
title: Trojan 协议中继器
---

# Trojan 协议中继器

> 源码位置: `I:/code/Prism/include/prism/protocol/trojan/relay.hpp`

## 概述

Trojan 协议中继器类，继承自 `transport::transmission`，提供协议握手、凭据验证和数据转发功能。Trojan 协议是一种基于 TLS 的加密代理协议，通过在应用层添加固定格式的头部来实现流量伪装和认证。

该类采用装饰器设计模式，透明地增强底层传输层的功能，支持链式组合。协议流程包括凭据读取、协议头部解析、格式验证、命令检查和数据转发。

## 类定义

```cpp
class relay : public psm::channel::transport::transmission, 
              public std::enable_shared_from_this<relay>
```

**继承关系**:
- `transport::transmission` - 统一传输层接口
- `std::enable_shared_from_this` - 安全共享指针管理

**设计特性**:
- 持有 `next_layer_` 独占所有权
- 支持 CONNECT（TCP 隧道）和 UDP_ASSOCIATE（UDP over TLS）
- 支持 IPv4、IPv6 和域名地址

## 构造函数

```cpp
explicit relay(shared_transmission next_layer, const config &cfg = {},
               std::function<bool(std::string_view)> credential_verifier = nullptr);
```

**参数**:
- `next_layer`: 底层传输层智能指针，必须已建立连接
- `cfg`: 协议配置
- `credential_verifier`: 用户凭据验证回调函数，可选

## 核心方法

### executor

获取关联的执行器。

```cpp
executor_type executor() const override;
```

### async_read_some / async_write_some

异步读写操作，透传到底层传输层。

```cpp
auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;

auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
    -> net::awaitable<std::size_t> override;
```

### close / cancel

关闭和取消操作。

```cpp
void close() override;
void cancel() override;
```

### handshake

执行 Trojan 协议握手，完整流程包括：

1. 读取 56 字节用户凭据并验证
2. 读取 CRLF 分隔符
3. 解析命令和地址类型
4. 读取目标地址和端口
5. 检查命令是否允许

```cpp
auto handshake() const -> net::awaitable<std::pair<fault::code, request>>;
```

**返回**: 握手结果和请求信息

### async_associate

处理 UDP_ASSOCIATE 命令，进入 UDP over TLS 模式。

```cpp
using route_callback = std::function<net::awaitable<std::pair<fault::code, net::ip::udp::endpoint>>(std::string_view, std::string_view)>;

auto async_associate(route_callback route_cb) const -> net::awaitable<fault::code>;
```

**流程**:
- 从 TLS 流读取封装的 UDP 数据包
- 解析目标地址后通过 UDP socket 转发
- 将响应封装回 TLS 流
- 支持空闲超时机制

### next_layer / release

获取底层传输层引用或释放所有权。

```cpp
psm::channel::transport::transmission &next_layer() noexcept;
const psm::channel::transport::transmission &next_layer() const noexcept;
shared_transmission release();
```

## 工厂函数

```cpp
using shared_relay = std::shared_ptr<relay>;

inline shared_relay make_relay(shared_transmission next_layer, const config &cfg = {},
                               std::function<bool(std::string_view)> credential_verifier = nullptr);
```

## 调用链

- [[core/protocol/trojan/format|Format]] - 使用解析函数处理协议头部
- [[core/protocol/trojan/constants|Constants]] - 命令和地址类型枚举
- [[core/protocol/trojan/config|Config]] - 配置结构
- [[core/channel/transport/transmission|Transmission]] - 底层传输层接口
- [[core/fault/code|Fault Code]] - 错误码处理