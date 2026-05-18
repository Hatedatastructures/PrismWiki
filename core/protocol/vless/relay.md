---
layer: core
source: "I:/code/Prism/include/prism/protocol/vless/relay.hpp"
---

# VLESS 协议中继器

> 源码位置: `I:/code/Prism/include/prism/protocol/vless/relay.hpp`

## 概述

VLESS 协议中继器类，提供协议握手和 UUID 验证功能。VLESS 协议运行在 TLS 内层，通过 UUID 进行用户认证。认证逻辑通过 verifier 回调委托给 pipeline 层，与 account::directory 对接，实现与其他协议统一的连接数限制和租约管理。

该类采用装饰器设计模式，透明地增强底层传输层的功能。

## 类定义

```cpp
class relay : public psm::channel::transport::transmission, 
              public std::enable_shared_from_this<relay>
```

**继承关系**:
- `transport::transmission` - 统一传输层接口
- `std::enable_shared_from_this` - 安全共享指针管理

## 构造函数

```cpp
explicit relay(shared_transmission next_layer, const config &cfg = {},
               std::function<bool(std::string_view)> verifier = nullptr);
```

**参数**:
- `next_layer`: 底层传输层智能指针
- `cfg`: VLESS 协议配置
- `verifier`: UUID 验证回调，接收 UUID 字符串返回是否认证通过。为 `nullptr` 时跳过认证（允许所有连接）

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

执行 VLESS 协议握手。

```cpp
auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
```

**流程**:
- 从传输层读取并解析 VLESS 请求头
- 通过 verifier 回调验证 UUID
- 发送响应
- 数据通过 preview 回放，读取即消费，不会残留

### async_associate

处理 UDP 命令，进入 UDP over TLS 模式。

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
                               std::function<bool(std::string_view)> verifier = nullptr);
```

## 调用链

- [[core/protocol/vless/format|Format]] - 使用解析函数处理协议头部
- [[core/protocol/vless/constants|Constants]] - 版本号和常量
- [[core/protocol/vless/config|Config]] - 配置结构
- [[core/channel/transport/transmission|Transmission]] - 底层传输层接口
- [[core/fault/code|Fault Code]] - 错误码处理
- [[core/agent/account/directory|Account Directory]] - UUID 认证对接