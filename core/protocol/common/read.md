---
layer: core
source: I:/code/Prism/include/prism/protocol/common/read.hpp
title: 共享 I/O 读取工具
tags: [protocol, common, read, io, coroutine, awaitable]
---

# 共享 I/O 读取工具

提供跨协议的通用读取辅助函数，封装从传输层批量读取数据的协程逻辑。

## 源码位置

`I:/code/Prism/include/prism/protocol/common/read.hpp`

## 命名空间

`psm::protocol::common`

## 函数定义

### read_min

批量读取至少指定数量的字节。

```cpp
[[nodiscard]] inline auto read_min(transport::transmission &transport,
                                    const std::span<std::byte> buffer,
                                    const std::size_t min_size)
    -> net::awaitable<std::pair<fault::code, std::size_t>>;
```

**流程**：
1. 循环调用 `async_read_some` 直到读取至少 `min_size` 字节
2. 遇到错误时返回 `fault::to_code(ec)` 和已读字节数
3. EOF (n == 0) 时返回 `fault::code::eof`

> **约束**: 若传输层返回部分数据后断开，`read_min` 返回已读字节数和 `eof` 错误码，而非继续等待。调用方必须检查实际读取字节数是否满足协议要求。

### remaining_opts

```cpp
struct remaining_opts
{
    transport::transmission &transport;
    std::span<std::byte> buffer;
    std::size_t current;   // 当前已读字节数
    std::size_t target;    // 目标字节数
};
```

将 `read_remaining` 的 4 个参数收敛到单结构体，符合 Rule 1（函数参数不超过 3 个）。

### read_remaining

精确补读剩余字节。

```cpp
[[nodiscard]] inline auto read_remaining(remaining_opts opts)
    -> net::awaitable<std::pair<fault::code, std::size_t>>;
```

从 `current` 位置继续读取，直到达到 `target` 字节。

## 设计决策

### 为什么提取为共享函数？

Trojan 和 VLESS 的 relay 都需要在握手阶段读取固定长度的协议头部（如 Trojan 的 56 字节凭据），然后读取剩余的变长部分（地址+端口）。提取 `read_min` 和 `read_remaining` 消除了两处完全相同的循环逻辑。

### 为什么 remaining_opts 是 struct 而非直接参数？

原始签名有 4 个参数（transport, buffer, current, target），违反 Rule 1（函数参数不超过 3 个）。收敛到 struct 是 Prism 的标准做法。

## 使用场景

- **Trojan relay::handshake**: `read_min` 读取 56 字节凭据 + 2 字节 CRLF，然后 `read_remaining` 读取地址数据
- **VLESS relay::handshake**: `read_min` 读取 26+ 字节请求头

## UDP over TLS 帧循环模板

`read_min` 也可用于 UDP over TLS 的帧循环中读取每个 UDP 帧，但实际的 UDP 帧循环使用 `common::udprelay` 中的 `frame_loop` 模板函数。

## 设计特点

- **纯协程**: 所有函数返回 `net::awaitable`
- **零拷贝**: 直接操作传入的缓冲区
- **错误码统一**: 使用 `fault::code` 系统

## 调用链

```
protocol/common::read_min -> transport.async_read_some
protocol/common::read_remaining -> transport.async_read_some
protocol/trojan/relay::handshake -> read_min(56) -> read_remaining
protocol/vless/relay::handshake -> read_min
```

## 依赖

- [[core/transport/transmission|Transmission]] - 传输层接口
- [[core/fault/code|Fault Code]] - 错误码系统
