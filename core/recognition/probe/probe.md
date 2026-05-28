---
tags: [recognition, probe]
layer: core
module: recognition
source: I:/code/Prism/include/prism/recognition/probe/probe.hpp
title: probe.hpp
---

# probe.hpp

外层协议探测，从传输层预读数据检测协议类型。

## 核心类型

### probe_result

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `protocol_type` | 检测到的协议类型 |
| `pre_read_data` | `array<byte, 32>` | 预读数据缓冲区 |
| `pre_read_size` | `size_t` | 实际预读数据大小 |
| `ec` | `fault::code` | 错误代码 |

方法：`success()` 检查成功（无错误且非 unknown）、`preload_view()` / `preload_bytes()` 获取预读数据视图。

## probe()

```cpp
auto probe(transport::transmission &transport, const std::size_t max_peek_size = 24)
    -> net::awaitable<probe_result>;
```

预读 max_peek_size 字节，调用 `detect()` 判断协议类型。预读数据保存到 `pre_read_data`，后续识别阶段可直接使用。

## 设计决策

### 为什么预读 24 字节？

24 字节覆盖所有目标协议的识别特征：TLS Record 头（5 字节）+ ClientHello 前几个固定字段，HTTP 方法名最长 8 字节（`OPTIONS `），SOCKS5 首字节即判定。既不会过度预读导致延迟，也避免预读不足无法识别。

**后果**: 缓冲区硬上限 32 字节（`pre_read_data` 大小），`max_peek_size` 超过 32 会被 clamp。

### 为什么预读后数据不回退？

`probe()` 使用 `async_read_some` 直接读取（非 peek 模式），数据被消费后不会回到 socket 缓冲区。预读数据保存在 `probe_result::pre_read_data` 中，后续识别阶段（ClientHello 解析）从这里获取而非重新读取。

**后果**: `probe_result` 必须传递到后续所有需要预读数据的阶段，不能丢弃。

## 约束

### transport 必须有数据可读

**类型**: 状态前置

**规则**: 调用 `probe()` 时 transport 上必须有数据到达。如果对端尚未发送数据，`async_read_some` 会挂起协程。

**违反后果**: 协程挂起直到对端发送数据或超时。如果是空连接（对端断开），返回 `eof` 错误。

**源码依据**: `probe.hpp:89-98`

### pre_read_data 引用不可跨协程挂起点

**类型**: 生命周期

**规则**: `preload_view()` / `preload_bytes()` 返回的视图指向 `probe_result` 内部缓冲区，`probe_result` 必须在视图使用期间存活。

**违反后果**: 悬挂指针。

**源码依据**: `probe.hpp:57-70`

## 协议类型

检测支持的协议类型（[[core/protocol/analysis|protocol_type]]）：

| 协议 | 首字节/特征 | 检测方式 |
|------|------------|----------|
| SOCKS5 | `0x05` | 首字节比较 |
| TLS | `0x16 0x03` | 前两字节比较 |
| HTTP | `GET/POST/...` | 方法名匹配 |
| Shadowsocks | 其他 | 排除法 fallback |
| Unknown | 空/无效 | 默认值 |

## 引用关系

### 依赖

- [[core/recognition/probe/analyzer|analyzer]]：detect() 函数
- [[core/channel/transport/transmission|transmission]]：传输层
- [[core/protocol/analysis|protocol_type]]：协议类型枚举
- [[core/fault/code|fault::code]]：错误码

### 被引用

- [[core/recognition/recognition|recognition]]：recognize() 中调用
