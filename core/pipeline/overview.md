---
title: "Pipeline 层 — 协议管道处理"
layer: core
source: "I:/code/Prism/include/prism/pipeline/"
created: 2026-05-17
updated: 2026-05-17
---

# Pipeline 层 — 协议管道处理

> 源码目录: `include/prism/pipeline/`
> 模块职责: 协议解析、帧编解码、双向转发

## 模块定位

Pipeline 层是 Prism 六阶段架构的第四阶段，位于 Dispatch 层和 Channel 层之间。

```
Dispatch 层 (处理器分发) -> Pipeline 层 (协议处理) -> Channel 层 (传输层)
```

## 核心职责

| 职责 | 描述 | 关键组件 |
|------|------|----------|
| 预读重放 | 包装预读数据，避免重复读取 | `preview` transport |
| 协议握手 | 执行协议级握手和认证 | `protocol::*::relay::handshake` |
| 目标解析 | 解析目标地址（域名/IP/端口） | `protocol::analysis::target` |
| 上游拨号 | 通过路由器或出站代理连接上游 | `dial()` |
| 隧道转发 | 建立双向数据流转发 | `tunnel()` |
| 多路复用 | 引导 mux 会话（smux/yamux） | `multiplex::bootstrap` |

## 目录结构

```
include/prism/pipeline/
├── primitives.hpp              # 管道原语定义（核心共享组件）
├── protocols.hpp               # 协议处理器聚合头文件
└── protocols/
    ├── http.hpp                # HTTP 代理处理
    ├── socks5.hpp              # SOCKS5 代理处理
    ├── trojan.hpp              # Trojan 协议处理
    ├── vless.hpp               # VLESS 协议处理
    └── shadowsocks.hpp         # SS2022 协议处理
```

## 组件详解

### primitives.hpp

管道原语定义，为所有协议处理器提供统一的基础功能。详见 [[core/pipeline/primitives|管道原语]]。

主要原语函数：
- `shut_close()` — 安全关闭传输层
- `dial()` — 上游拨号（路由器/出站代理两种路径）
- `ssl_handshake()` — TLS 服务端握手
- `preview` — 预读数据回放包装器
- `tunnel()` — 双向隧道转发
- `forward()` — 拨号 + 隧道组合

### 协议处理器

每种协议有独立的处理函数，由 Dispatch 层调用：

| 处理器 | 函数 | TLS 层 | 认证方式 |
|--------|------|--------|----------|
| [[core/pipeline/protocols/http|HTTP]] | `http()` | 外层可选 | Basic Auth 或无 |
| [[core/pipeline/protocols/socks5|SOCKS5]] | `socks5()` | 无 | 方法协商 |
| [[core/pipeline/protocols/trojan|Trojan]] | `trojan()` | Session 层 | SHA224 密码哈希 |
| [[core/pipeline/protocols/vless|VLESS]] | `vless()` | Session 层 | UUID |
| [[core/pipeline/protocols/shadowsocks|SS2022]] | `shadowsocks()` | 外层 ShadowTLS | PSK |

## 处理器统一签名

所有协议处理器遵循统一的函数签名：

```cpp
auto handler(session_context &ctx, std::span<const std::byte> data)
    -> net::awaitable<void>;
```

参数说明：
- `ctx`: 会话上下文，包含入站传输、配置、内存资源
- `data`: 预读数据，协议检测时读取的初始数据

## 数据流向

```
Dispatch 层输出 (session_context + protocol_type)
       │
       ▼
Pipeline 层入口 (handler 函数)
       │
       ├── wrap_with_preview() 包装入站传输
       │
       ├── protocol::*::relay 创建中继器
       │
       ├── relay::handshake() 执行握手
       │
       ├── analysis::resolve() 解析目标地址
       │
       ├── 命令分发
       │     ├── TCP: dial() + tunnel()
       │     ├── UDP: make_datagram_router() + async_associate()
       │     ├── mux: bootstrap()
       │
       ▼
Channel 层 (传输层抽象)
```

## 相关文档

- [[core/pipeline/primitives|管道原语详解]]
- [[core/pipeline/protocols/http|HTTP 协议管道]]
- [[core/pipeline/protocols/socks5|SOCKS5 协议管道]]
- [[core/pipeline/protocols/trojan|Trojan 协议管道]]
- [[core/pipeline/protocols/vless|VLESS 协议管道]]
- [[core/pipeline/protocols/shadowsocks|SS2022 协议管道]]