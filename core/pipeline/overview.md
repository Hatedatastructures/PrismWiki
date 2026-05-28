---
title: "Pipeline 层 -- 协议管道处理"
layer: core
module: pipeline
source: I:/code/Prism/include/prism/pipeline/
created: 2026-05-17
updated: 2026-05-27
tags: [pipeline, overview, protocol-handler, dispatch, tunnel]
---

# Pipeline 层 — 协议管道处理

Pipeline 层是 Prism 六阶段架构的第四阶段，位于 Dispatch 层和 Channel 层之间。核心职责：预读重放、协议握手、目标解析、上游拨号、双向转发。

## 设计决策

### 为什么每个协议是独立函数而非继承基类？

协议处理器的逻辑差异大（HTTP 解析请求行 vs SOCKS5 解析握手帧 vs Trojan 读 hex 密码），无法抽象为统一接口。独立函数 + `switch` 分发更直接，编译器可内联优化。Dispatch 层通过 `protocol_type` 枚举选择对应函数。

**后果**: 新增协议只需添加新函数和 switch case，不涉及继承层次。

## 模块组成

| 组件 | 文件 | 说明 |
|------|------|------|
| [[core/pipeline/primitives\|primitives]] | `primitives.hpp` | 管道原语（shut_close/dial/tunnel/forward） |
| [[core/pipeline/protocols/http\|http]] | `protocols/http.hpp` | HTTP 代理处理 |
| [[core/pipeline/protocols/socks5\|socks5]] | `protocols/socks5.hpp` | SOCKS5 代理处理 |
| [[core/pipeline/protocols/trojan\|trojan]] | `protocols/trojan.hpp` | Trojan 协议处理 |
| [[core/pipeline/protocols/vless\|vless]] | `protocols/vless.hpp` | VLESS 协议处理 |
| [[core/pipeline/protocols/shadowsocks\|shadowsocks]] | `protocols/shadowsocks.hpp` | SS2022 协议处理 |
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