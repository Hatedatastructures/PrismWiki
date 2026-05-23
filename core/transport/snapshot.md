---
layer: core
source: prism/transport/snapshot.hpp
module: "transport"
title: Snapshot 可回滚传输
tags: [transport, snapshot, 回滚, 装饰器, TLS伪装]
created: 2026-05-23
---

# Snapshot 可回滚传输

> 命名空间: `psm::transport`
> 源码: `include/prism/transport/snapshot.hpp`

## 概述

`snapshot` 是可回滚的传输层装饰器，自动捕获所有从内层传输读取的字节到内部缓冲区，支持 `rewind()` 将读取位置归零重新读取。主要用于 TLS 伪装方案的依次尝试：每个 scheme 读取的数据被 snapshot 捕获，失败时 rewind，下一个 scheme 从同一起点重试。

## 设计约束

| 约束 | 说明 |
|------|------|
| rewind 仅在未发生写入时有效 | `wrote_ == false` |
| 认证阶段是纯读取 | 安全 rewind |
| 一旦开始写入 | transport 状态不可恢复 |

## 类定义

```cpp
class snapshot final : public transmission
{
public:
    explicit snapshot(shared_transmission inner,
                      memory::resource_pointer mr = memory::current_resource());

    // transmission 接口
    auto transport_type() const noexcept -> type override;
    auto next_layer() noexcept -> transmission * override;
    auto executor() const -> executor_type override;

    // 读取（带捕获和回放）
    auto async_read_some(std::span<std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;

    // 写入（标记已写入，禁止 rewind）
    auto async_write_some(std::span<const std::byte> buffer, std::error_code &ec)
        -> net::awaitable<std::size_t> override;

    // 回滚控制
    void rewind() noexcept;
    auto can_rewind() const noexcept -> bool;

    // 传输控制
    void close() override;
    void cancel() override;

    // 装饰器导航
    auto inner() const noexcept -> shared_transmission;

private:
    shared_transmission inner_;
    memory::vector<std::byte> captured_;
    std::size_t read_pos_{0};
    bool wrote_{false};
};
```

## 核心机制

### 两阶段读取

```
async_read_some(buffer, ec)
     │
     ▼
  read_pos_ < captured_.size() ?
     │                    │
   是 │                 否 │
     ▼                    ▼
  ┌──────────────────┐  ┌──────────────────────────┐
  │ Phase 1: 回放    │  │ Phase 2: 从内层读取 + 捕获│
  │                  │  │                          │
  │ memcpy 到 buffer │  │ n = co_await inner_->    │
  │ read_pos_ += n   │  │   async_read_some(buf,ec)│
  │ ec = {}          │  │                          │
  │ 返回 n           │  │ captured_.insert(end,     │
  │ (同步完成)       │  │   buf.data(), buf.data()+n)│
  └──────────────────┘  │ read_pos_ += n           │
                        │ 返回 n                   │
                        └──────────────────────────┘
```

**Phase 1 (回放)**: 从 `captured_` 缓冲区复制数据到用户 buffer，同步完成，不挂起协程。

**Phase 2 (捕获)**: 从内层传输异步读取数据，同时将读取的数据追加到 `captured_` 缓冲区。

### 写入与 rewind 限制

```
async_write_some(buffer, ec)
     │
     ▼
  wrote_ = true          // 标记已写入，禁止 rewind
     │
     ▼
  co_await inner_->async_write_some(buffer, ec)
```

一旦调用 `async_write_some`，`wrote_` 被设置为 `true`，`can_rewind()` 返回 `false`。这是因为写入操作会改变传输层的内部状态（如 TLS 握手状态），回滚无法恢复这些状态。

### rewind 操作

```
rewind()
     │
     ▼
  read_pos_ = 0       // 归零读取位置
  // captured_ 数据保留，不清空
```

`rewind()` 仅将 `read_pos_` 归零，不清空 `captured_`。下次 `async_read_some` 将从 `captured_` 起点回放所有已捕获的数据。

## 工厂函数

```cpp
inline auto make_snapshot(shared_transmission inner,
                           memory::resource_pointer mr = memory::current_resource())
    -> shared_transmission;
```

创建 snapshot 包装器，返回 `shared_transmission`。

## 使用场景

### TLS 伪装方案依次尝试

```
                    make_snapshot(inbound)
                           │
                           ▼
                    ┌──────────────────┐
                    │  snapshot         │
                    │  captured_ = {}   │
                    │  read_pos_ = 0    │
                    │  wrote_ = false   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        Scheme A        Scheme B        Scheme C
        (reality)       (shadowtls)     (restls)
              │              │              │
        读取数据        rewind()        rewind()
        写入握手        从头读取        从头读取
              │              │              │
        失败 ↓          失败 ↓          成功 ↓
        rewind()        rewind()        继续使用
              │              │              │
              └──────────────┴──────────────┘
```

```cpp
// 实际代码示例
auto snap = transport::make_snapshot(std::move(inbound));

for (auto &scheme : schemes) {
    auto result = co_await scheme->execute(*snap);

    if (result.success) {
        // 方案成功，继续使用 snap
        break;
    }

    if (snap->can_rewind()) {
        snap->rewind();  // 回滚，下一个方案从同一起点开始
    } else {
        // 无法回滚，所有方案失败
        break;
    }
}
```

### 与 preview 配合

在 [[core/transport/preview|preview]] 之上包装 snapshot：

```cpp
// 先包装预读数据
auto wrapped = transport::wrap_with_preview(inbound, preread_data);

// 再包装 snapshot 用于方案回滚
auto snap = transport::make_snapshot(std::move(wrapped));
```

## 内存管理

`captured_` 使用 PMR `memory::vector<std::byte>` 分配：

- 数据累积增长，随读取量增加
- 通常在 TLS 握手阶段捕获几百字节到几 KB
- 使用 PMR 内存资源，与 [[core/memory/overview|内存管理]] 集成
- 方案成功后 snapshot 可被丢弃，释放 captured_ 内存

## 性能特征

| 操作 | 开销 | 说明 |
|------|------|------|
| 构造 | O(1) | 仅初始化成员 |
| Phase 1 读取 | O(N) memcpy | 从 captured_ 复制 |
| Phase 2 读取 | O(N) memcpy + 异步 I/O | 追加到 captured_ |
| 写入 | O(1) + 异步 I/O | 设置 wrote_ 标志 |
| rewind | O(1) | 归零 read_pos_ |
| 内存占用 | O(总读取量) | captured_ 累积 |

## 相关页面

- [[core/transport/overview|transport — 传输层总览]]
- [[core/transport/preview|preview — 预读数据回放]]
- [[core/transport/transmission|transmission — 传输抽象]]
- [[core/stealth/overview|stealth — TLS 伪装]]
- [[core/recognition|recognition — 协议识别]]
- [[core/memory/overview|memory — PMR 内存管理]]
