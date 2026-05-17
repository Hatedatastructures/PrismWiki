---
layer: core
source: I:/code/Prism/include/prism/multiplex/yamux/frame.hpp
---

# yamux::frame - yamux 协议帧格式定义

## 源码位置

`I:/code/Prism/include/prism/multiplex/yamux/frame.hpp`

## 概述

定义 yamux 多路复用协议的帧格式、消息类型、标志位和编解码函数。兼容 Hashicorp/yamux 协议规范。

## 协议常量

```cpp
constexpr std::uint8_t protocol_version = 0x00;  // yamux 规范规定 Version 固定为 0
constexpr std::size_t frame_header_size = 12;     // 帧头大小
constexpr std::uint32_t initial_stream_window = 256 * 1024;  // 256KB
```

## 帧头结构

```cpp
struct frame_header
{
    std::uint8_t version = protocol_version;     // 协议版本
    message_type type = message_type::data;      // 消息类型
    flags flag = flags::none;                    // 标志位
    std::uint32_t stream_id = 0;                 // 流标识符
    std::uint32_t length = 0;                    // 长度字段

    bool is_session() const noexcept;  // 检查是否为会话级消息
};
```

帧格式：`[Version 1B][Type 1B][Flags 2B BE][StreamID 4B BE][Length 4B BE]`

## 消息类型

```cpp
enum class message_type : std::uint8_t
{
    data = 0x00,           // 数据帧
    window_update = 0x01,  // 窙口更新帧
    ping = 0x02,           // 心跳帧
    go_away = 0x03         // 会话终止帧
};
```

## 标志位

```cpp
enum class flags : std::uint16_t
{
    none = 0x0000,  // 无标志
    syn = 0x0001,   // SYN 同步标志
    ack = 0x0002,   // ACK 确认标志
    fin = 0x0004,   // FIN 半关闭标志
    rst = 0x0008    // RST 重置标志
};
```

### 标志位语义

| 消息类型 + 标志 | 语义 |
|----------------|------|
| Data + SYN | 携带地址数据的新流创建 |
| Data + FIN | 半关闭流 |
| Data + RST | 强制重置流 |
| WindowUpdate + SYN | 打开新流 |
| WindowUpdate + ACK | 确认流创建 |
| Ping + SYN | 心跳请求 |
| Ping + ACK | 心跳响应 |

## GoAway 原因码

```cpp
enum class go_away_code : std::uint32_t
{
    protocol_error = 1  // 协议错误
};
```

## Length 字段含义

| 消息类型 | Length 含义 |
|----------|-------------|
| Data | 载荷字节数 |
| WindowUpdate | 窗口增量 |
| Ping | ping 标识符 |
| GoAway | 终止原因码 |

## 编解码函数

```cpp
// 编码帧头
std::array<std::byte, frame_header_size> build_header(const frame_header &hdr);

// 解析帧头
std::optional<frame_header> parse_header(std::span<const std::byte> buffer);

// 构建 WindowUpdate 帧
std::array<std::byte, frame_header_size> build_window_update_frame(
    flags f, std::uint32_t stream_id, std::uint32_t delta);

// 构建 Ping 帧
std::array<std::byte, frame_header_size> build_ping_frame(
    flags f, std::uint32_t ping_id);

// 构建 GoAway 帧
std::array<std::byte, frame_header_size> build_go_away_frame(go_away_code code);

// 构建 FIN 帧
std::array<std::byte, frame_header_size> make_fin_frame(std::uint32_t stream_id);
```

## data_frame 结构

```cpp
struct data_frame
{
    std::array<std::byte, frame_header_size> header{};  // 帧头
    std::vector<std::byte> payload;                     // 载荷
};
```

## 帧构建函数

```cpp
// 构建 Data 帧
data_frame make_data_frame(flags f, std::uint32_t stream_id,
                           std::span<const std::byte> payload);

// 构建 SYN 帧
data_frame make_syn_frame(std::uint32_t stream_id,
                          std::span<const std::byte> payload);
```

## 标志位辅助函数

```cpp
// 按位与运算
flags operator&(flags a, flags b);

// 检查标志位
bool has_flag(flags f, flags flag);
```

## 关联文档

- [[core/multiplex/yamux/craft|yamux::craft]] - yamux 协议实现
- [[core/multiplex/yamux/config|yamux::config]] - yamux 协议配置