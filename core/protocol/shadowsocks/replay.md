---
layer: core
source: "I:/code/Prism/include/prism/protocol/shadowsocks/replay.hpp"
---

# SS2022 PacketID 重放检测

> 源码位置: `I:/code/Prism/include/prism/protocol/shadowsocks/replay.hpp`

## 概述

WireGuard 风格 PacketID 滑动窗口重放检测。SS2022 (SIP022) UDP 使用 PacketID 进行重放保护。采用 WireGuard 式滑动窗口：维护最近 N 个 PacketID 的位图，拒绝窗口外的旧包和重复包。

## 命名空间

```cpp
namespace psm::protocol::shadowsocks
```

## 类定义

### replay_window

PacketID 滑动窗口重放过滤器。

```cpp
class replay_window
```

**特性**:
- 窗口大小 64
- 支持 64 位 PacketID 空间
- 线程不安全，每个 UDP 会话独立持有

## 常量

```cpp
static constexpr std::size_t window_size = 64;
```

窗口大小固定为 64。

## 核心方法

### check_and_update

检查并更新 PacketID。

```cpp
auto check_and_update(std::uint64_t packet_id) -> bool;
```

**参数**:
- `packet_id`: 收到的 PacketID（大端序转本地后的值）

**返回**: `true` 表示首次出现（可接受），`false` 表示重放或太旧

## 实现逻辑

```
1. 首次使用：任何值都接受
   - 设置 base_ = packet_id
   - 清空位图，设置 bit 0

2. 在窗口右侧（packet_id >= base_ + window_size）：
   - 移动窗口
   - 如果完全跳过窗口，重置位图
   - 否则部分移动位图
   - 设置新 base_ 和最高位

3. 在窗口内部（packet_id >= base_）：
   - 检查位图对应位
   - 已设置表示重复，返回 false
   - 未设置则设置并返回 true

4. 在窗口左侧（packet_id < base_）：
   - 太旧，返回 false
```

## 内部成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `base_` | `std::uint64_t` | 窗口基线（最新接受的 PacketID - window_size + 1） |
| `bitmap_` | `std::bitset<window_size>` | 窗口位图 |
| `initialized_` | `bool` | 是否已初始化 |

## 算法特点

**WireGuard 式设计**:
- 高效：位图操作，O(1) 复杂度
- 简单：无复杂状态机
- 安全：拒绝窗口外的所有旧包

**适用场景**:
- UDP 会话独立持有
- 无需线程同步
- 适合高频小包场景

## 调用链

- [[core/protocol/shadowsocks/relay|Relay]] - UDP 处理中使用重放检测
- [[core/protocol/shadowsocks/constants|Constants]] - PacketID 长度定义
- [[core/protocol/shadowsocks/salts|Salts]] - TCP Salt 重放保护（类似功能）