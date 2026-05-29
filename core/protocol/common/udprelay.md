---
title: 共享 UDP 中继
created: 2026-05-25
updated: 2026-05-27
layer: core
source: include/prism/protocol/common/udprelay.hpp
tags: [protocol, common, udp, relay, frame-loop, tls]
---

# 共享 UDP 中继

协议无关的 UDP 数据报中继基础设施，被 Trojan 和 VLESS 的 UDP over TLS 实现共用。

> 源码: include/prism/protocol/common/udprelay.hpp | 实现: inline header-only

## 命名空间

`psm::protocol::common`

## 核心类型

### loop_cfg

UDP 帧循环配置，聚合定时器和参数。

```cpp
struct loop_cfg
{
    net::steady_timer &idle_timer;      // 空闲超时计时器
    std::string_view log_tag;           // 日志标签
    std::uint32_t idle_timeout;         // 空闲超时时间（秒）
    std::uint32_t max_datagram;         // 最大数据报长度
    traffic_callback on_traffic{nullptr}; // 流量通知回调（函数指针+void*，避免堆分配）
    void *traffic_ctx{nullptr};         // 回调上下文
};
```

### udp_buffers

UDP 会话缓冲区集合，使用 PMR 分配器避免热路径堆分配。

```cpp
struct udp_buffers
{
    memory::vector<std::byte> recv;     // 接收缓冲区
    memory::vector<std::byte> send;     // 发送缓冲区
    memory::vector<std::byte> response; // 响应缓冲区
};
```

### frame_ctx（模板结构）

`frame_loop` 回调参数聚合。将解析函数、构建函数、路由回调收敛到单结构体，符合 Rule 1（函数参数不超过 3 个）。

### resp_ctx（模板结构）

`send_frame` 参数聚合。将 sender 端点、响应长度、构建函数、日志标签收敛到单结构体。

## 核心函数

### relay_packet

转发 UDP 数据包到目标并接收响应。

```cpp
inline auto relay_packet(relay_opts opts)
    -> net::awaitable<std::tuple<fault::code, std::size_t, net::ip::udp::endpoint>>;
```

延迟打开 socket：首次调用时按目标协议族打开，后续复用同一 socket。

### send_frame

通过 TLS 传输层发送 UDP 帧响应。

```cpp
template <typename BuildFn, typename UdpFrame>
auto send_frame(transport::transmission &transport,
                const resp_ctx<BuildFn, UdpFrame> &ctx,
                udp_buffers &buf) -> net::awaitable<bool>;
```

根据 sender_ep 的地址族自动选择 IPv4/IPv6 地址类型，构建协议帧后写入 TLS 流。

### frame_loop

UDP over TLS 帧循环（模板函数）。

```cpp
template <typename ParseFn, typename BuildFn, typename UdpFrame, typename UdpParseResult>
auto frame_loop(transport::transmission &tls_transport,
                frame_ctx<...> frame_ctx,
                loop_cfg config) -> net::awaitable<void>;
```

## frame_loop 内部流程

```
┌─────────────────────────────────────────────────────┐
│  frame_loop 主循环                                    │
│                                                      │
│  1. 重置空闲定时器                                    │
│  2. 并发等待: TLS 读取 || 空闲超时                    │
│     ├── 超时 → 流量回调 → 返回                        │
│     └── 读取成功 ↓                                    │
│  3. parse_fn 解析 UDP 帧 → 提取目标地址和 payload      │
│  4. route_cb 路由目标地址 → 获取 UDP endpoint          │
│  5. relay_packet 发送 UDP → 接收响应                   │
│  6. send_frame 封装响应 → 写回 TLS 流                  │
│  7. 累计流量统计 → 回到步骤 1                          │
└─────────────────────────────────────────────────────┘
```

> **约束**: `frame_loop` 使用 `awaitable_operators::operator||` 实现读取与超时的并发等待。定时器在每次读取成功后立即取消。若 TLS 读取失败或 EOF，循环终止并调用流量回调。

## 设计决策

### 为什么 frame_loop 是模板函数？

Trojan 和 VLESS 的 UDP 帧格式不同（Trojan 有 Length+CRLF，VLESS 无）。但帧循环的核心逻辑相同：读帧 -> 解析 -> 转发 -> 接收响应 -> 编码 -> 写回 TLS。模板参数化解析/构建函数避免了虚函数开销和代码重复。

**后果**: 编译器为每个协议生成特化版本的帧循环，零运行时开销。

### 为什么 relay_packet 延迟打开 socket？

首次调用时按目标地址的协议族（IPv4/IPv6）打开 UDP socket，后续复用。这避免了在不知道目标地址时盲目打开 socket（可能打开错误的协议族）。

**后果**: 首个 UDP 数据报有 socket 创建延迟，后续数据报直接发送。

### 为什么用函数指针 + void* 而非 std::function？

`traffic_callback` 使用 `void(*)(void*, uint64_t, uint64_t)` 而非 `std::function`。在 UDP 热路径上，`std::function` 的类型擦除会引入堆分配（若捕获超过小缓冲区大小）。函数指针 + void* 是零开销的回调方案。

## 调用链

- [[core/protocol/trojan/relay|Trojan relay::async_associate]] -> frame_loop
- [[core/protocol/vless/relay|VLESS relay::async_associate]] -> frame_loop

## 依赖

- [[core/protocol/trojan/relay|Trojan relay]]
- [[core/protocol/vless/relay|VLESS relay]]
- [[core/transport/transmission|transmission]]
- [[core/protocol/common/address|address]]
- [[core/protocol/common/target|target]]
