---
layer: core
source: I:/code/Prism/include/prism/protocol/common/read.hpp
---

# 共享 I/O 读取工具

提供跨协议的通用读取辅助函数，封装从传输层批量读取数据的协程逻辑。

## 源码位置

`I:/code/Prism/include/prism/protocol/common/read.hpp`

## 函数定义

### read_at_least

批量读取至少指定数量的字节。

```cpp
auto read_at_least(channel::transport::transmission &transport,
                    const std::span<std::byte> buffer,
                    const std::size_t min_size)
    -> net::awaitable<std::pair<fault::code, std::size_t>>;
```

**流程**：
1. 循环调用 `async_read_some` 直到读取至少 `min_size` 字节
2. 遇到错误或 EOF 时提前返回
3. 返回已读取字节数和错误码

**错误处理**：
- 系统错误 -> `fault::to_code(ec)`
- EOF (n == 0) -> `fault::code::eof`

### read_remaining

精确补读剩余字节。

```cpp
auto read_remaining(channel::transport::transmission &transport,
                     const std::span<std::byte> buffer,
                     std::size_t current,
                     const std::size_t target)
    -> net::awaitable<std::pair<fault::code, std::size_t>>;
```

**流程**：
1. 从 `current` 位置继续读取
2. 直到达到 `target` 字节

## 使用场景

被 Trojan 和 VLESS relay 共同使用，消除重复代码：

- 读取固定长度的协议头部
- 读取剩余的请求/响应数据

## 设计特点

- **纯协程**：所有函数返回 `net::awaitable`
- **零拷贝**：直接操作传入的缓冲区
- **错误码统一**：使用 `fault::code` 系统

## 调用链

```
protocol/common::read_at_least -> transport.async_read_some
protocol/common::read_remaining -> transport.async_read_some
```

## 依赖

- [[channel/transport/transmission]] - 传输层接口
- [[log/fault/code]] - 错误码系统