---
title: "bootstrap — 多路复用会话引导"
source: "include/prism/multiplex/bootstrap.hpp"
implementation: "src/prism/multiplex/bootstrap.cpp"
module: "multiplex"
type: api
tags: [multiplex, bootstrap, sing-mux, 协商, 协议分流]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/core|core]]"
  - "[[multiplex/config|config]]"
  - "[[multiplex/smux/craft|smux::craft]]"
  - "[[multiplex/yamux/craft|yamux::craft]]"
  - "[[channel/transport/transmission|transmission]]"
  - "[[resolve/router|router]]"
---

# bootstrap.hpp

> 源码: `include/prism/multiplex/bootstrap.hpp`
> 实现: `src/prism/multiplex/bootstrap.cpp`
> 模块: [[multiplex/config|multiplex]]

## 概述

多路复用会话引导。统一入口，完成 sing-mux 协议协商后根据客户端选择的协议类型创建对应的 [[multiplex/core|core]] 子类实例。

协商格式：
- 基本格式（Version==0）：`[Version 1B][Protocol 1B]`
- 扩展格式（Version>0）：`[Version 1B][Protocol 1B][PaddingLen 2B BE][Padding N bytes]`

Protocol 字段：0=smux, 1=yamux。协商完成后 transport 上的后续数据由具体 mux 协议帧解释。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[multiplex/core|core]] | 返回 shared_ptr<core> |
| 依赖 | [[multiplex/config|config]] | 多路复用配置 |
| 依赖 | [[multiplex/smux/craft|smux::craft]] | 创建 smux 会话实例 |
| 依赖 | [[multiplex/yamux/craft|yamux::craft]] | 创建 yamux 会话实例 |
| 依赖 | [[channel/transport/transmission|transmission]] | 已建立的传输层连接 |
| 依赖 | [[resolve/router|router]] | 路由器引用 |
| 被依赖 | agent/session | session 层调用 bootstrap |

## 命名空间

`psm::multiplex`

---

## 函数: bootstrap

**功能说明**

引导多路复用会话。执行 sing-mux 协议协商，根据客户端 Protocol 字段选择 smux 或 yamux 协议创建对应实例。调用者通过 core 基类指针操作，无需关心具体协议类型。

**签名**

```cpp
[[nodiscard]] auto bootstrap(channel::transport::shared_transmission transport, resolve::router &router,
                             const config &cfg, memory::resource_pointer mr = memory::current_resource())
    -> net::awaitable<std::shared_ptr<core>>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `shared_transmission` | 已建立的传输层连接 |
| `router` | `router&` | 路由器引用，用于解析地址并连接目标 |
| `cfg` | `const config&` | 多路复用配置 |
| `mr` | `resource_pointer` | 内存资源，默认使用当前 PMR 资源 |

**返回值**

`net::awaitable<std::shared_ptr<core>>` — mux 会话实例的共享指针，协商失败时返回 nullptr。

**调用（向下）**

- `negotiate()` — 内部函数，执行 sing-mux 协议协商读取 Version/Protocol 字段
- `smux::craft` 构造函数 — 当 Protocol=0 时创建 smux 会话
- `yamux::craft` 构造函数 — 当 Protocol=1 时创建 yamux 会话

**被调用（向上）**

- `session` 层 — 在识别出 mux 协议后调用 bootstrap 创建会话

**知识域**

sing-mux 协议协商、工厂模式、协程返回值、多态创建。

---

## 内部函数: negotiate

**功能说明**

执行 sing-mux 协议协商。从 transport 读取 sing-mux 协议头并消费。Protocol 字段指示客户端选择的多路复用协议类型（0=smux, 1=yamux）。支持 Version>0 的扩展格式（含 padding）。

**签名**

```cpp
auto negotiate(transmission &transport, const memory::resource_pointer mr)
    -> net::awaitable<std::pair<std::error_code, protocol_type>>;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `transport` | `transmission&` | 已建立的传输层连接 |
| `mr` | `resource_pointer` | PMR 内存资源，用于 padding 缓冲区分配 |

**返回值**

`net::awaitable<std::pair<std::error_code, protocol_type>>` — 协商结果对：成功时 error_code 为空，失败时携带错误码。

**调用（向下）**

- `transport.async_read()` — 读取协议头和 padding 数据

**被调用（向上）**

- `bootstrap()` — 内部调用 negotiate 获取协议类型

**知识域**

sing-mux 协议格式、大端序解析、错误码传播。

---

## 协议分流流程

```
bootstrap()
  → negotiate() 读取 [Version 1B][Protocol 1B]
  → 解析 Protocol 字段
  → switch(protocol):
      case yamux → make_shared<yamux::craft>(...)
      case smux  → make_shared<smux::craft>(...)
  → 返回 shared_ptr<core> 多态指针
```
