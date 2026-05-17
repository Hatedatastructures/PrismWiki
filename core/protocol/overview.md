---
layer: core
source: I:/code/Prism/include/prism/protocol
---

# 协议层概览

协议层是 Prism 的核心组件之一，负责处理各种代理协议的解析、握手和中继逻辑。

## 模块结构

协议层按功能划分为三个子模块：

### common - 共享组件

跨协议通用定义，包括地址类型、传输形态和 I/O 工具函数。

- [[core/protocol/common/address]] - IPv4/IPv6/域名地址类型
- [[core/protocol/common/form]] - TCP/UDP 传输形态枚举
- [[core/protocol/common/read]] - 批量读取辅助函数

### http - HTTP 代理

HTTP 代理协议实现，支持 CONNECT 隧道和正向代理。

- [[core/protocol/http/parser]] - 请求头解析和 Basic 认证
- [[core/protocol/http/relay]] - HTTP 代理中继器

### socks5 - SOCKS5 协议

完整的 SOCKS5 服务端实现，支持 TCP 隧道和 UDP 中继。

- [[core/protocol/socks5/constants]] - 协议常量定义
- [[core/protocol/socks5/config]] - 协议配置结构
- [[core/protocol/socks5/wire]] - 线级报文解析
- [[core/protocol/socks5/stream]] - SOCKS5 中继器

## 设计原则

协议层遵循以下设计原则：

1. **零拷贝友好**：解析结果直接指向原始缓冲区，避免额外内存分配
2. **协程原生**：所有 I/O 操作使用 `net::awaitable`，无缝集成 Boost.Asio
3. **类型安全**：使用 `std::variant` 和枚举类实现多态，避免裸指针和魔数
4. **错误码统一**：使用 `fault::code` 系统，便于错误追踪和处理

## 调用链

协议层被 [[agent/worker/worker]] 调用，处理入站连接的握手和路由决策：

```
agent/front/listener -> agent/worker/worker -> protocol::*::relay -> channel/transport/transmission
```