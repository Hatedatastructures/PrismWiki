---
title: "config — 多路复用通用配置"
source: "include/prism/multiplex/config.hpp"
module: "multiplex"
type: api
tags: [multiplex, config, 配置, 协议选择]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/smux/config|smux::config]]"
  - "[[multiplex/yamux/config|yamux::config]]"
  - "[[multiplex/core|core]]"
  - "[[multiplex/bootstrap|bootstrap]]"
---

# config.hpp

> 源码: `include/prism/multiplex/config.hpp`
> 模块: [[multiplex/core|multiplex]]

## 概述

多路复用通用配置。定义多路复用层的协议选择和全局开关。各协议的完整配置参数分别定义在对应子目录的 config.hpp 中（[[multiplex/smux/config|smux::config]]、[[multiplex/yamux/config|yamux::config]]）。协议类型由 sing-mux 协商动态决定，无需配置。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[multiplex/smux/config|smux::config]] | smux 协议配置 |
| 依赖 | [[multiplex/yamux/config|yamux::config]] | yamux 协议配置 |
| 被依赖 | [[multiplex/core|core]] | core 构造函数接收 config 引用 |
| 被依赖 | [[multiplex/bootstrap|bootstrap]] | bootstrap 函数接收 config 引用 |
| 被依赖 | [[multiplex/smux/craft|smux::craft]] | 读取 smux 子配置 |
| 被依赖 | [[multiplex/yamux/craft|yamux::craft]] | 读取 yamux 子配置 |

## 命名空间

`psm::multiplex`

---

## 枚举: protocol_type

**功能说明**

多路复用协议类型枚举，用于 sing-mux 协商后选择具体实现。

**签名**

```cpp
enum class protocol_type : std::uint8_t {
    smux  = 0,  // xtaci/smux v1 + sing-mux 协商
    yamux = 1   // Hashicorp/yamux + sing-mux 协商
};
```

**参数**

无（枚举类型）。

**返回值**

枚举值，用于 `bootstrap::negotiate()` 的协议选择 switch。

**调用（向下）**

无。

**被调用（向上）**

- `bootstrap::negotiate()` — 解析 Protocol 字段后转换为 protocol_type
- `bootstrap::bootstrap()` — 根据 protocol_type 创建对应实例

**知识域**

sing-mux 协议规范、枚举类型安全。

---

## 结构体: config

**功能说明**

多路复用配置入口，聚合全局开关和各协议独立配置。

**签名**

```cpp
struct config {
    bool enabled = false;
    smux::config smux;
    yamux::config yamux;
};
```

**字段**

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | `bool` | `false` | 是否启用多路复用服务端 |
| `smux` | `smux::config` | 见 [[multiplex/smux/config|smux 配置]] | smux 协议配置 |
| `yamux` | `yamux::config` | 见 [[multiplex/yamux/config|yamux 配置]] | yamux 协议配置 |

**返回值**

无（结构体定义）。

**调用（向下）**

无。

**被调用（向上）**

- agent 配置加载 — 从 JSON 反序列化 config
- [[multiplex/core|core]] 构造函数 — 存储 config 引用
- [[multiplex/bootstrap|bootstrap]] — 传递 config 到子类构造函数

**知识域**

配置聚合模式、子配置嵌套、JSON 反序列化。
