---
layer: core
source: prism/transport/preview.hpp
module: "transport"
title: Preview 预读数据回放
tags: [transport, preview, 预读回放, 装饰器, 协议嗅探]
created: 2026-05-23
---

# Preview 预读数据回放

> 命名空间: `psm::transport`
> 源码: `include/prism/transport/preview.hpp`

## 概述

`preview` 是传输层装饰器，用于在协议嗅探阶段后回放已预读的数据。在协议识别过程中，部分数据可能已从入站传输中读取（如预读 24 字节进行协议探测）。`preview` 将这些预读数据保存在内部缓冲区中，在后续读取时优先返回预读数据，待预读数据耗尽后再委托给内层传输，确保协议管道在嗅探后仍能一致地处理数据流。

## 类定义

```cpp
class preview final : public transmission
{
public:
    explicit preview(shared_transmission inner,
                     std::span<const std::byte> preread,
                     memory::resource_pointer mr = memory::current_resource());

    // transmission 接口实现
    auto transport_type() const noexcept -> type override;
    auto executor() const -> executor_type override;
    auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;
    auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;

    // Completion-handler 风格重载
    void async_read_some(std::span<std::byte> buffer,
                         net::any_completion_handler<void(boost::system::error_code, std::size_t)> handler) override;
    void async_write_some(std::span<const std::byte> buffer,
                          net::any_completion_handler<void(boost::system::error_code, std::size_t)> handler) override;

    // 完整写入
    auto async_write(std::span<const std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t>;

    void close() override;
    void cancel() override;

    // 装饰器导航
    auto next_layer() noexcept -> transmission * override;
    auto inner() const noexcept -> shared_transmission;

private:
    shared_transmission inner_;
    memory::vector<std::byte> preread_buffer_;
    std::size_t offset_{0};
};
```

## 核心机制

### 读取流程

```
async_read_some(buffer, ec)
     │
     ▼
  offset_ < preread_buffer_.size() ?
     │                    │
   是 │                 否 │
     ▼                    ▼
  从 preread_buffer_     从 inner_ 读取
  复制到 buffer          追加到 captured_
  offset_ += n           返回读取字节数
  ec = {}
  返回 n                 (异步 co_await)
     │
     ▼
  (同步完成，不挂起协程)
```

**关键**: 预读数据阶段是同步完成的，不涉及异步 I/O。只有预读数据耗尽后才进行真正的异步读取。

### 写入流程

写入操作直接委托给内层传输，不缓冲：

```
async_write_some(buffer, ec)
     │
     ▼
  inner_->async_write_some(buffer, ec)
```

### 关闭与取消

- `close()`: 清空预读缓冲区后关闭内层传输
- `cancel()`: 取消内层传输上所有挂起的异步操作

## 工厂函数

### wrap_with_preview

```cpp
inline auto wrap_with_preview(shared_transmission inbound,
                               std::span<const std::byte> data,
                               memory::resource_pointer mr = memory::current_resource())
    -> shared_transmission;
```

将入站传输包装为带预读数据的传输。若 `data` 为空则直接返回原始入站传输，不创建装饰器。

**效果**: 在 [[core/recognition/overview|recognition]] 模块探测协议后，将预读的 24 字节（或更多）数据包装回传输层，使后续协议处理无需关心预读逻辑。

## 装饰器链

`preview` 是传输层装饰器链的一环，与 [[core/transport/snapshot|snapshot]] 共同工作：

```
原始传输 (reliable/tcp)
     │
     ├── 被 snapshot 包装 (用于 TLS 伪装方案回滚)
     │        │
     │        └── 被 preview 包装 (用于协议嗅探后回放)
     │
     └── 被 preview 包装 (用于协议嗅探后回放)
```

解包时使用 [[core/connect/overview|connect::util]] 中的 `peel_to_raw()` 函数穿透所有装饰层。

## 使用场景

### 协议识别后回放

```cpp
// recognition 模块探测阶段
std::array<std::byte, 24> probe_buffer{};
auto [ec, n] = co_await inbound->async_read_some(probe_buffer, ec);

// 根据探测结果识别协议
auto detected = detect_protocol(probe_buffer);

// 将预读数据包装回传输层
auto wrapped = transport::wrap_with_preview(
    std::move(inbound),
    std::span<const std::byte>(probe_buffer.data(), n)
);

// 后续协议处理从 wrapped 读取时，先返回预读数据，再读取新数据
```

### 与 snapshot 配合

```cpp
// 在 stealth 模块中，snapshot 包装在 preview 之上
auto snap = transport::make_snapshot(std::move(wrapped));

// scheme 尝试
co_await scheme->execute(*snap);

// 失败时回滚
if (snap->can_rewind()) {
    snap->rewind();
    // 下一个 scheme 从同一起点开始
}
```

## 内存管理

`preview` 构造时将预读数据复制到内部 `memory::vector<std::byte>`，确保数据所有权安全：

- 预读数据可能与原始缓冲区共享生命周期
- 复制到 PMR vector 后，原始缓冲区可安全释放
- 使用 PMR 内存资源分配，与 [[core/memory/overview|内存管理]] 集成

## 性能特征

| 操作 | 开销 | 说明 |
|------|------|------|
| 构造 | O(N) 复制 | N 为预读数据长度 |
| 预读阶段读取 | O(N) memcpy | 同步完成，不挂起协程 |
| 内层读取 | 0 额外开销 | 直接委托 |
| 写入 | 0 额外开销 | 直接委托 |
| 内存占用 | N 字节 | N 为预读数据长度（通常 24-256 字节） |

## 设计决策

### 为什么 preview 需要存在？

**问题**: 协议识别（recognition）阶段需要预读入站数据来判断协议类型（HTTP/SOCKS5/TLS/SS2022），但这些数据属于后续协议处理器的输入。预读后数据已从 socket 缓冲区移出，后续读取无法重新获取。

**选择**: preview 装饰器将预读数据保存在内部缓冲区，后续 async_read_some 优先返回预读数据，耗尽后委托给内层传输。这使协议管道在嗅探后仍能一致地处理完整数据流。

**后果**: 预读数据回放对协议处理器完全透明。代价是额外的内存拷贝（通常 24 字节）和一次间接调用。

**替代方案**: 让每个协议处理器自己处理"已预读 N 字节"的参数，增加所有处理器的复杂度。

**源码依据**: `transport/preview.hpp:1-181`

### 为什么 preview 覆写 completion-handler 路径？

**问题**: 当 encrypted 的 ssl::stream 内部调用底层传输的 async_read_some 时，如果 preview 的预读数据阶段通过 co_spawn 桥接，会增加一个协程帧。

**选择**: preview 覆写 completion-handler 路径的 async_read_some，在预读数据阶段同步完成（memcpy + 立即调用 handler），不发起任何异步操作。

**后果**: TLS 握手期间的预读数据回放零协程帧开销。

**源码依据**: `transport/preview.hpp:111`

### 为什么 wrap_with_preview 对空数据直接返回原始传输？

**问题**: 如果协议检测阶段预读 0 字节（如连接立即关闭），创建空的 preview 装饰器是无意义的开销。

**选择**: wrap_with_preview 检查 data 是否为空，空则直接返回原始 inbound，不创建装饰器。

**后果**: 减少不必要的装饰器层级，降低装饰器链深度。

**源码依据**: `transport/preview.hpp:171-179`

## 约束

### 预读数据构造时复制

**类型**: 所有权安全
**规则**: preview 构造时将预读数据复制到内部 `memory::vector<std::byte>`。原始缓冲区在复制后可安全释放。
**违反后果**: 如果不复制而是保存 span，原始缓冲区可能在 preview 生命周期内被释放

### offset_ 单调递增不回退

**类型**: 状态一致性
**规则**: offset_ 从 0 单调递增到 preread_buffer_.size()，没有回退机制。预读数据只能被消费一次。
**违反后果**: 回退 offset_ 会导致数据重复消费

## 相关页面

- [[core/transport/overview|transport — 传输层总览]]
- [[core/transport/snapshot|snapshot — 可回滚传输]]
- [[core/transport/transmission|transmission — 传输抽象]]
- [[core/recognition/overview|recognition — 协议识别]]
- [[core/stealth/overview|stealth — TLS 伪装]]
- [[core/connect/overview|connect/util — 连接工具函数]]
