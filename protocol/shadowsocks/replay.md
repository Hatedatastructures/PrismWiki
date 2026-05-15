---
title: "replay — SS2022 PacketID 滑动窗口重放检测"
source: "include/prism/protocol/shadowsocks/replay.hpp"
module: "protocol"
type: api
tags: [protocol, shadowsocks, replay, 重放检测, 滑动窗口, WireGuard, PacketID]
related:
  - "[[protocol/shadowsocks/salts|salts]]"
  - "[[protocol/shadowsocks/tracker|tracker]]"
  - "[[protocol/shadowsocks/constants|constants]]"
created: 2026-05-15
updated: 2026-05-15
---

# replay.hpp

> 源码: `include/prism/protocol/shadowsocks/replay.hpp`
> 模块: [[protocol|protocol]] > shadowsocks

## 概述

WireGuard 风格 PacketID 滑动窗口重放检测。SS2022 (SIP022) UDP 使用 PacketID 进行重放保护。采用 WireGuard 式滑动窗口：维护最近 N 个 PacketID 的位图，拒绝窗口外的旧包和重复包。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[protocol/shadowsocks/tracker|tracker]] | UDP 会话跟踪器使用重放窗口 |

## 命名空间

`psm::protocol::shadowsocks`

---

## 类: replay_window

> 源码: `include/prism/protocol/shadowsocks/replay.hpp:24`

### 概述

PacketID 滑动窗口重放过滤器。窗口大小 64，支持 64 位 PacketID 空间。线程不安全，每个 UDP 会话独立持有。

### 常量

| 常量 | 类型 | 值 | 说明 |
|------|------|----|------|
| `window_size` | `std::size_t` | `64` | 滑动窗口大小（位数） |

---

### 函数: check_and_update()

#### 功能说明

检查 PacketID 是否重复，并更新滑动窗口。首次使用时任何值都接受。窗口逻辑：
- `packet_id >= base_ + window_size`：在窗口右侧，移动窗口
- `packet_id >= base_`：在窗口内部，检查对应位是否已设置
- `packet_id < base_`：在窗口左侧（太旧），拒绝

#### 签名

```cpp
auto check_and_update(std::uint64_t packet_id) -> bool;
```

#### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `packet_id` | `std::uint64_t` | 待检查的 PacketID（大端序转本地后的值） |

#### 返回值

`bool` — `true` 表示首次出现（可接受），`false` 表示重放或太旧。

#### 调用（向下）

- `std::bitset::test()` / `set()` / `operator>>=` — 位图操作

#### 被调用（向上）

- [[protocol/shadowsocks/tracker|tracker]] UDP 会话处理

#### 知识域

滑动窗口算法、WireGuard 重放保护

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| `base_` | `std::uint64_t` | 窗口基址（最小有效 PacketID） |
| `bitmap_` | `std::bitset<64>` | 窗口位图 |
| `initialized_` | `bool` | 是否已初始化 |

## 相关页面

- [[protocol/shadowsocks/salts|salts]] — Salt 重放保护池
- [[protocol/shadowsocks/tracker|tracker]] — UDP 会话跟踪器
- [[protocol/shadowsocks/constants|constants]] — 协议常量
