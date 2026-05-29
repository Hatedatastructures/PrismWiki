---
layer: core
source: "include/prism/protocol"
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

## 设计决策（WHY）

### 为什么所有协议共用统一的 handle() 入口签名

每个协议处理器暴露 `handle(context::session&, span<const byte>)` 统一入口。这使得 session::diversion() 可以通过函数指针表分发，无需虚函数或 variant 访问者模式。预读数据通过 span 直接传入，避免二次读取。新增协议只需实现同名函数即可接入分发表。

### 为什么 SS2022 relay 保持活跃而 Trojan/VLESS 释放传输层

SS2022 的 AEAD 流加密在整个会话生命周期内持续加解密每个数据块，relay 必须持有加密状态。Trojan/VLESS 仅在握手阶段使用传输层读取认证帧，完成后将底层传输释放给 tunnel 进行裸转发。这是流加密协议与帧认证协议的根本架构差异。

### 为什么跨协议共享 address 类型

SOCKS5、Trojan、VLESS、Shadowsocks 四个协议都有相同的 IPv4/IPv6/域名地址编码需求。提取到 common/address.hpp 消除四处重复定义，使用 std::variant 实现类型安全多态，std::visit 编译期分发。address 结构设计为 POD 友好，可直接从协议缓冲区零拷贝填充。

### 为什么 Trojan/VLESS relay 继承 transport::transmission

采用装饰器模式，Trojan/VLESS 的 conn 类继承 transmission 抽象基类，将协议帧解析叠加在传输层之上。这使得 relay 可以像普通传输层一样被 tunnel 消费，协议处理与数据转发解耦。SS2022 同样使用此模式，但持续持有传输层所有权。

### 为什么 TLS 解析放在协议层而非识别层

TLS ClientHello 解析（hello.hpp/record.hpp）是识别层（recognition）和伪装层（stealth）的共享基础设施。放在协议层是因为其本质是协议报文解析，与 HTTP/SOCKS5 解析同属协议特征提取范畴。识别层和伪装层通过引用协议层的 tls 子模块使用。

## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| 协程纯度 | 禁止在 handle() 中使用阻塞操作 | 阻塞 worker 线程，影响同线程所有连接 | 协程约定 |
| PMR 内存 | 热路径容器使用 memory::string/vector | 堆分配开销破坏延迟保证 | PMR 策略 |
| 错误码 | 热路径使用 fault::code，不抛异常 | 异常栈展开开销 + 协程生命周期风险 | 双轨错误处理 |
| 入口签名 | handle(session&, span<byte>) 统一 | 无法接入分发表 | 分发层契约 |
| 零拷贝 | 解析结果指向原始缓冲区 | 额外内存拷贝增加延迟 | 设计原则 |
| 传输层释放 | Trojan/VLESS 握手后释放 transmission | 资源泄漏或 use-after-free | 生命周期约束 |

## 故障场景

### 1. SS2022 AEAD 解密失败

AEAD 密文被篡改或密钥不匹配时，解密返回认证失败。relay 关闭连接，但不泄露密钥信息。由于 SS2022 是首包加密，首个数据包解密失败即判定为非法连接。

### 2. Trojan/VLESS 认证失败

Trojan SHA224 密码不匹配或 VLESS UUID 不匹配时，handler 直接关闭连接。不发送错误响应，避免向探测者暴露协议特征。认证失败在 handshake 阶段完成，不进入 tunnel。

### 3. SOCKS5 握手版本不匹配

客户端发送非 0x05 版本号时，wire 解析立即拒绝。SOCKS5 不支持的方法列表（如仅支持 NO_AUTH 但客户端要求 GSSAPI）也会导致握手失败并返回 0xFF 方法选择。

### 4. HTTP 请求畸形报文

parser 无法解析请求行或头部时，返回 400 Bad Request 响应。CONNECT 方法要求完整的 host:port 格式，缺失端口导致解析失败。Basic 认证失败返回 407 Proxy Authentication Required。

### 5. 预读数据丢失

probe 预读 24 字节后传入 handle()，如果协议实现忽略预读数据而重新读取，会导致数据流错位。各协议 handler 必须消费完整的预读缓冲区后才切换到流式读取。

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| context::session | instance -> protocol | session 提供路由器、传输层、配置等上下文，handler 不持有所有权 |
| connect::forward() | protocol -> connect | 所有协议的 TCP 隧道转发共用 forward()，传入 target 和 inbound |
| transport::transmission | protocol <- transport | Trojan/VLESS/SS2022 的 conn 继承 transmission，装饰器模式叠加协议帧 |
| protocol::target | protocol -> connect | 解析后的目标地址桥接结构，connect::router 根据 positive 标志选择路由 |
| protocol::common::address | protocol 内部共享 | SOCKS5/Trojan/VLESS/SS2022 共享地址类型，零拷贝解析 |
| protocol::tls::* | protocol -> recognition/stealth | TLS 解析结果供识别层特征分析和伪装层方案选择使用 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 说明 |
|------|----------|------|
| handle() 签名变更 | instance/session 分发表、所有协议 handler | 核心接口，破坏性变更 |
| protocol_type 枚举增删 | recognition/probe、session/diversion | 影响协议探测和分发逻辑 |
| target 结构字段变更 | connect/router、所有协议 handler | 路由决策依赖 target.positive 标志 |
| common/address 类型变更 | SOCKS5/Trojan/VLESS/SS2022 全部受影响 | 共享基础类型 |

### 对内影响

| 变更 | 影响范围 | 说明 |
|------|----------|------|
| 新增协议子模块 | 需实现 process.hpp + conn.hpp + config.hpp + framing.hpp | 遵循现有模板 |
| framing 帧格式变更 | 仅影响对应协议的编解码 | 模块隔离良好 |
| tls 解析逻辑变更 | recognition 特征分析、stealth 方案执行 | 跨模块共享，需回归测试 |
| common/read 工具变更 | 所有协议的批量读取逻辑 | 低频变更，影响面广 |
