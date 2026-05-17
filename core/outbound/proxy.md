---
title: "proxy — 出站代理抽象接口"
layer: core
source: "I:/code/Prism/include/prism/outbound/proxy.hpp"
module: "outbound"
type: interface
tags: [outbound, proxy, interface, adapter]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/outbound/direct
  - core/channel/transport/transmission
  - core/protocol/analysis
  - core/fault/code
---

# proxy — 出站代理抽象接口

> 源码位置: `I:/code/Prism/include/prism/outbound/proxy.hpp`
> 模块: [[core/outbound/overview|outbound]]

## 接口定位

`proxy` 是所有出站协议（direct、socks5、http、trojan、vless、ss 等）以及代理组（url_test、fallback、load_balance、selector）必须实现的核心接口。该接口采用纯协程设计，使用 `net::awaitable` 作为异步操作返回类型，通过组合模式使代理组和单个代理共享同一接口。

**等价于 mihomo 的 `constant/adapters.go` 中 `ProxyAdapter` 接口。**

## 类型定义

### datagram_router_fn

```cpp
/// UDP 数据报路由回调类型
using datagram_router_fn = std::function<net::awaitable<std::pair<fault::code,
                                                  net::ip::udp::endpoint>>(std::string_view, std::string_view)>;
```

UDP 数据报路由回调函数类型，接受 `(host, port)` 参数，返回 `(错误码, udp_endpoint)` 对。用于将 DNS 解析封装在实现内部，避免直接传递 router 引用。

### shared_transmission

```cpp
using shared_transmission = channel::transport::shared_transmission;
```

传输层智能指针类型，用于管理传输层对象的生命周期。详见 [[core/channel/transport/transmission|transmission]]。

## 接口定义

```cpp
namespace psm::outbound
{
    namespace net = boost::asio;
    using shared_transmission = channel::transport::shared_transmission;

    using datagram_router_fn = std::function<net::awaitable<std::pair<fault::code,
                                                      net::ip::udp::endpoint>>(std::string_view, std::string_view)>;

    class proxy
    {
    public:
        virtual ~proxy() = default;

        /// 建立 TCP 连接到目标
        virtual auto async_connect(const protocol::analysis::target &target,
                                   const net::any_io_executor &executor)
            -> net::awaitable<std::pair<fault::code, shared_transmission>> = 0;

        /// 创建 UDP 数据报路由回调
        virtual auto make_datagram_router()
            -> datagram_router_fn = 0;

        /// 获取代理名称
        [[nodiscard]] virtual auto name() const -> std::string_view = 0;

        /// 是否支持 UDP（默认返回 true）
        [[nodiscard]] virtual auto supports_udp() const -> bool { return true; }
    };
}
```

## 核心方法详解

### async_connect()

```cpp
virtual auto async_connect(const protocol::analysis::target &target,
                           const net::any_io_executor &executor)
    -> net::awaitable<std::pair<fault::code, shared_transmission>> = 0;
```

建立 TCP 连接到目标地址，是出站代理的核心方法。

**参数**:
- `target`: 目标地址信息，包含 host、port 和 positive 标记
  - `host`: 目标主机名或 IP 地址
  - `port`: 目标端口
  - `positive`: 正向/反向代理标记
- `executor`: 用于创建连接的执行器

**返回值**: 协程对象，完成后返回 `(错误码, 传输对象)` 对

**实现策略**:
| 实现类 | 路由策略 |
|--------|----------|
| `direct` | 直连走 DNS 解析 + 连接池 |
| 协议代理 | 走上游协议握手 |
| 代理组 | 走子代理选择 |

---

### make_datagram_router()

```cpp
virtual auto make_datagram_router()
    -> datagram_router_fn = 0;
```

创建 UDP 数据报路由回调函数。

**返回值**: 回调函数，接受 `(host, port)` 返回 `(错误码, udp_endpoint)`

**设计决策**:
- 替代直接传递 router 引用
- 将 DNS 解析封装在实现内部
- 支持不同代理的 UDP 路由策略

---

### name()

```cpp
[[nodiscard]] virtual auto name() const -> std::string_view = 0;
```

获取代理名称，用于日志和调试。

**返回值**: 代理名称的字符串视图

---

### supports_udp()

```cpp
[[nodiscard]] virtual auto supports_udp() const -> bool { return true; }
```

是否支持 UDP 协议。不支持 UDP 的代理应重写此方法返回 `false`。

**返回值**: 默认返回 `true`

## 设计模式

### 组合模式

代理组通过组合模式委托给子代理：

```
┌─────────────────┐
│  ProxyGroup     │ ─── 实现 proxy 接口
│  (url_test)     │
└─────────────────┘
         │
         │ 委托
         ▼
┌─────────────────┐     ┌─────────────────┐
│  Proxy (trojan) │     │  Proxy (ss)     │ ─── 实现 proxy 接口
└─────────────────┘     └─────────────────┘
```

调用方无需区分单个代理和代理组：

```cpp
// 透明调用
auto [ec, trans] = co_await proxy->async_connect(target, executor);
```

### 策略模式

不同代理实现不同的路由策略：

| 实现 | async_connect 行为 |
|------|-------------------|
| `direct` | DNS 解析 → 直连 |
| `socks5` | SOCKS5 握手 → 上游连接 |
| `trojan` | TLS + Trojan 协议 |
| `url_test` | 选择延迟最低的子代理 |

## 调用链

```
proxy::async_connect
    │
    ├─→ direct::async_connect
    │       → router::async_forward (DNS + 连接池)
    │       → make_reliable(传输层)
    │
    ├─→ trojan::async_connect
    │       → 上游代理 async_connect
    │       → TLS 握手
    │       → Trojan 协议封装
    │
    └─→ ProxyGroup::async_connect
            → 选择子代理
            → 子代理::async_connect
```

## target 结构

```cpp
namespace protocol::analysis
{
    struct target
    {
        std::string_view host;    // 目标主机名或 IP
        std::string_view port;    // 目标端口
        bool positive;            // true: 正向代理, false: 反向代理
    };
}
```

详见 [[core/protocol/analysis|protocol::analysis]]。

## 参见

- [[core/outbound/direct|direct]] — 直连出站代理实现
- [[core/channel/transport/transmission|transmission]] — 传输层抽象接口
- [[core/protocol/analysis|protocol::analysis]] — 协议分析模块
- [[core/fault/code|fault::code]] — 错误码定义
- [[core/resolve/router|router]] — 分发层路由器