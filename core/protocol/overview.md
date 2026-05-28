---
layer: core
source: "I:/code/Prism/include/prism/protocol"
title: 协议层概览
tags: [protocol, overview, core, dispatch, handler]
---

# 协议层概览

协议层是 Prism 的核心组件之一，负责处理各种代理协议的解析、握手和中继逻辑。

## 模块结构

协议层按功能划分为多个子模块：

### common - 共享组件

跨协议通用定义，包括地址类型、传输形态和 I/O 工具函数。

- [[core/protocol/common/address|Address]] - IPv4/IPv6/域名地址类型
- [[core/protocol/common/form|Form]] - TCP/UDP 传输形态枚举
- [[core/protocol/common/framing|Framing]] - 跨协议共享地址/端口线级解析
- [[core/protocol/common/read|Read]] - 批量读取辅助函数
- [[core/protocol/common/mux|Mux]] - 多路复用标记地址检测
- [[core/protocol/common/target|Target]] - 目标地址桥梁结构
- [[core/protocol/common/udprelay|UDP Relay]] - 协议无关 UDP 中继基础设施

### http - HTTP 代理

HTTP 代理协议实现，支持 CONNECT 隧道和正向代理。

- [[core/protocol/http/parser|Parser]] - 请求头解析和 Basic 认证
- [[core/protocol/http/relay|Relay]] - HTTP 代理中继器

### socks5 - SOCKS5 协议

完整的 SOCKS5 服务端实现（RFC 1928），支持 TCP 隧道和 UDP 中继。

- [[core/protocol/socks5/constants|Constants]] - 协议常量定义
- [[core/protocol/socks5/config|Config]] - 协议配置结构
- [[core/protocol/socks5/wire|Wire]] - 线级报文解析
- [[core/protocol/socks5/stream|Stream]] - SOCKS5 中继器

### trojan - Trojan 协议

基于 TLS 的加密代理协议，SHA224 密码认证。

- [[core/protocol/trojan/constants|Constants]] - 命令和地址类型枚举
- [[core/protocol/trojan/config|Config]] - 协议配置结构
- [[core/protocol/trojan/format|Format]] - 协议帧编解码（含完整字节级图解）
- [[core/protocol/trojan/relay|Relay]] - Trojan 中继器

### vless - VLESS 协议

基于 TLS 的轻量代理协议，UUID 认证，支持多路复用。

- [[core/protocol/vless/constants|Constants]] - 版本号、命令和地址类型
- [[core/protocol/vless/config|Config]] - 协议配置结构
- [[core/protocol/vless/format|Format]] - 协议帧编解码（含完整字节级图解）
- [[core/protocol/vless/relay|Relay]] - VLESS 中继器

### shadowsocks - SS2022 协议

Shadowsocks 2022 (SIP022) AEAD 加密代理协议。

- [[core/protocol/shadowsocks/constants|Constants]] - AEAD 加密常量
- [[core/protocol/shadowsocks/config|Config]] - 协议配置（PSK、加密方法）
- [[core/protocol/shadowsocks/format|Format]] - AEAD 帧结构全景图
- [[core/protocol/shadowsocks/relay|Relay]] - AEAD 流加密中继器
- [[core/protocol/shadowsocks/salts|Salts]] - Salt 重放保护池
- [[core/protocol/shadowsocks/replay|Replay]] - UDP PacketID 滑动窗口

### tls - TLS 分析

TLS ClientHello 解析和特征提取，为识别层和伪装层提供共享基础设施。

- [[core/protocol/tls/types|Types]] - TLS 记录层常量和特征结构
- [[core/protocol/tls/signal|Signal]] - ClientHello 解析器
- [[core/protocol/tls/feature_bitmap|Feature Bitmap]] - 特征位图快速匹配

## 设计原则

1. **零拷贝友好**: 解析结果直接指向原始缓冲区，避免额外内存分配
2. **协程原生**: 所有 I/O 操作使用 `net::awaitable`，无缝集成 Boost.Asio
3. **类型安全**: 使用 `std::variant` 和枚举类实现多态，避免裸指针和魔数
4. **错误码统一**: 使用 `fault::code` 系统，便于错误追踪和处理
5. **装饰器模式**: Trojan/VLESS relay 继承 `transport::transmission`，可链式组合

> **约束**: SS2022 的 relay 在整个会话生命周期内保持活跃（持续 AEAD 加解密），而 Trojan/VLESS relay 在握手完成后释放底层传输层给 tunnel。

## 协议处理流程

```
listener 接受连接 -> balancer 选择 worker -> launch 创建 session
    |
session 调用 recognition::recognize():
    probe(24B) -> 检测 HTTP/SOCKS5/TLS/SS2022
    | (仅 TLS)
    ClientHello -> 特征分析 -> 方案执行
    |
session::diversion() 分发到协议处理器:
    http/socks5/trojan/vless/shadowsocks handler
    |
handler 通过 connect::dial 建立上游 -> tunnel 双向转发
```

## 调用链

协议层被 [[core/instance/worker/worker|worker]] 调用，处理入站连接的握手和路由决策：

```
instance/front/listener -> instance/worker/worker -> protocol::*::relay -> connect/tunnel/forward
```
