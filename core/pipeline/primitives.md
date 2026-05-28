---
title: "primitives — 管道原语"
layer: core
source: "I:/code/Prism/include/prism/connect/"
created: 2026-05-17
updated: 2026-05-27
tags: [pipeline, primitives, dial, tunnel, forward]
---

# primitives — 管道原语

协议管道共享的通用原语组件。为 HTTP、SOCKS5、Trojan、VLESS、SS2022 等具体协议处理提供底层支撑。

> **注意**: wiki 历史版本引用 `pipeline/primitives.hpp`，实际源码分散在 connect/、transport/ 模块中。下表标注了每个原语的实际源码位置。

## 原语清单

| 原语 | 实际源码 | 说明 |
|------|----------|------|
| `shut_close()` | `connect/util.hpp` | 安全关闭传输对象（裸指针/智能指针重载） |
| `dial(router, ...)` | `connect/dial/dial.hpp` | 通过路由器拨号（正向/反向） |
| `dial(proxy, ...)` | `outbound/direct.hpp` | 通过出站代理拨号 |
| `ssl_handshake()` | `instance/worker/tls.hpp` | TLS 服务端握手 |
| `preview` | `transport/preview.hpp` | 预读数据回放包装器 |
| `wrap_with_preview()` | `transport/preview.hpp` | 将入站传输包装为带预读数据的传输 |
| `tunnel()` | `connect/tunnel/tunnel.hpp` | 双向隧道转发 |
| `forward()` | `connect/tunnel/forward.hpp` | 拨号 + 隧道组合 |
| `find_reliable()` | `connect/util.hpp` | 从装饰链中解包找到 reliable 传输 |

## 设计决策

### 为什么 tunnel() 任一方向断开即终止？

TCP 连接是全双工的，但代理场景中任一方向断开通常意味着远端关闭。保持另一个方向打开没有意义（对方不会再发数据），反而占用资源。`operator||` 并行执行两个方向，第一个完成时另一个被取消。

**后果**: 隧道关闭时两端传输都被 `shut_close()`。

### 为什么 forward() 是自由函数而非成员？

`forward()` 组合了拨号（dial）和隧道（tunnel），横跨 connect 和 transport 两个模块。作为自由函数避免了对特定类的依赖，协议处理器直接调用。

**后果**: `forward()` 的 `ctx` 参数是非 const 引用，因为它会移动 `ctx.inbound`。

### 为什么 preview 继承 transmission？

`preview` 需要替代原始传输对象的位置，上层代码通过 `transmission` 接口读写，不感知预读回放。继承 `transmission` 保证类型兼容。

**后果**: `find_reliable()` 需要 `dynamic_cast` 穿透 preview/snapshot 装饰层才能拿到底层 TCP socket。

## 约束

### shut_close() 幂等性

**类型**: 状态前置

**规则**: `shut_close()` 对空指针/空 shared_ptr 是空操作。对已关闭的传输，`shutdown_write()` 和 `close()` 应安全重复调用。

**违反后果**: 如果底层传输的 `close()` 不是幂等的，重复调用可能触发错误。

**源码依据**: `connect/util.hpp`

### wrap_with_preview() 移动 ctx.inbound

**类型**: 调用顺序

**规则**: `wrap_with_preview()` 会 `std::move(ctx.inbound)`，调用后 `ctx.inbound` 为空。

**违反后果**: 调用后通过 `ctx.inbound` 访问传输对象将得到空指针。

**源码依据**: `transport/preview.hpp`

## 引用关系

### 被引用

- [[core/pipeline/protocols/http|HTTP]]：使用 `wrap_with_preview` + `forward`
- [[core/pipeline/protocols/socks5|SOCKS5]]：使用 `wrap_with_preview` + `forward` + UDP 路由
- [[core/pipeline/protocols/trojan|Trojan]]：使用 `wrap_with_preview` + `forward`
- [[core/pipeline/protocols/vless|VLESS]]：使用 `wrap_with_preview` + `forward`
- [[core/pipeline/protocols/shadowsocks|SS2022]]：使用 `wrap_with_preview` + `forward`
