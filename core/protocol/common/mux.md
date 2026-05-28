---
title: 多路复用目标检测
created: 2026-05-25
updated: 2026-05-27
layer: core
source: I:/code/Prism/include/prism/protocol/common/mux.hpp
tags: [protocol, common, mux, sing-box, multiplex]
---

# 多路复用目标检测

检测目标地址是否为 mux 多路复用标记地址。

> 源码: include/prism/protocol/common/mux.hpp | 实现: inline header-only

## 命名空间

`psm::protocol`

## 核心类型

### mux_switch

```cpp
enum class mux_switch : std::uint8_t
{
    off,  // 禁用多路复用
    on    // 启用多路复用
};
```

## 核心函数

### is_mux_target

```cpp
[[nodiscard]] inline auto is_mux_target(std::string_view host, mux_switch mux) noexcept
    -> bool;
```

检测目标主机名是否以 `.mux.sing-box.arpa` 结尾（Mihomo/sing-box 兼容标记地址）。

## 设计决策

### is_mux_target 判断逻辑

函数仅做两件事：检查 `mux_switch` 是否为 `on`，以及 host 是否以 `.mux.sing-box.arpa` 后缀结尾。这是 Mihomo/sing-box 生态的约定标记地址——客户端在需要多路复用时将目标地址改为该特殊域名，服务端据此判断是否进入 mux 分支。

判断为 `true` 时，handler 会将真实目标地址从 mux 会话中提取（而非直接使用该标记域名），并走 `multiplex::smux`/`yamux`/`h2mux` 路径而非普通 dial。

> **约束**: 任何不在 sing-box 兼容生态中的客户端无法触发 mux 路径。这符合 Prism 作为 sing-box 协议兼容实现的定位。若需支持其他 mux 标记方案，需在此函数中扩展检测逻辑。

## 使用场景

- **Trojan handler**: 解析 Trojan 请求后检查目标地址是否为 mux 标记，决定走普通隧道还是多路复用
- **VLESS handler**: VLESS 的 mux 命令 (0x7F) 也会触发多路复用路径，但此处通过地址检测处理客户端使用标记地址的情况

## 调用链

- [[core/pipeline/protocols/trojan|trojan handler]] -> is_mux_target
- [[core/pipeline/protocols/vless|vless handler]] -> is_mux_target
