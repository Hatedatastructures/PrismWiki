---
title: "frame — yamux 帧格式定义与编解码"
source: "include/prism/multiplex/yamux/frame.hpp"
implementation: "src/prism/multiplex/yamux/frame.cpp"
module: "multiplex"
type: api
tags: [multiplex, yamux, frame, 帧格式, 编解码]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[multiplex/yamux/craft|yamux::craft]]"
  - "[[multiplex/yamux/config|yamux::config]]"
  - "[[multiplex/smux/frame|smux::frame]]"
---

# frame.hpp — yamux 帧格式定义与编解码

> 源码: `include/prism/multiplex/yamux/frame.hpp`
> 实现: `src/prism/multiplex/yamux/frame.cpp`
> 模块: [[multiplex/yamux/craft|multiplex]] > yamux

## 概述

yamux 帧格式定义与编解码。兼容 Hashicorp/yamux 协议规范。

帧格式为 12 字节定长帧头，所有多字节字段使用大端序（网络字节序）：
`[Version 1B][Type 1B][Flags 2B BE][StreamID 4B BE][Length 4B BE]`

消息类型与 Length 字段的对应关系：Data 中 Length 为载荷字节数，WindowUpdate 中 Length 为窗口增量，Ping 中 Length 为 ping 标识符，GoAway 中 Length 为错误码。

与 [[multiplex/smux/frame|smux]] 的 8 字节小端帧头不同，yamux 使用 12 字节大端帧头，并引入完整的流量控制（窗口管理）和标志位系统。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[multiplex/yamux/craft|yamux::craft]] | 帧编解码 |

## 命名空间

`psm::multiplex::yamux`

---

## 常量

| 常量 | 值 | 说明 |
|------|----|------|
| `protocol_version` | `0x00` | 版本号固定为 0 |
| `frame_header_size` | `12` | 帧头大小 |
| `initial_stream_window` | `262144` (256KB) | 初始流窗口 |

---

## 枚举: message_type

**功能说明**

yamux 帧消息类型，对应帧头的 Type 字段（1 字节）。

**签名**

```cpp
enum class message_type : std::uint8_t {
    data         = 0x00,  // 数据帧
    window_update = 0x01, // 窗口更新帧
    ping         = 0x02,  // 心跳帧
    go_away      = 0x03   // 会话终止帧
};
```

**参数**

无（枚举类型）。

**返回值**

枚举值，用于帧头的 Type 字段。

**调用（向下）**

无。

**被调用（向上）**

- `yamux::craft::frame_loop()` — 根据消息类型分发
- `yamux::craft::push_frame()` — 编码帧头时设置类型

**知识域**

yamux 协议消息类型、Hashicorp/yamux 兼容性。

---

## 枚举: flags

**功能说明**

yamux 帧标志位，对应帧头的 Flags 字段（2 字节大端序）。标志位可组合使用，不同消息类型下语义不同。

**签名**

```cpp
enum class flags : std::uint16_t {
    none = 0x0000,  // 无标志
    syn  = 0x0001,  // SYN 同步标志
    ack  = 0x0002,  // ACK 确认标志
    fin  = 0x0004,  // FIN 半关闭标志
    rst  = 0x0008   // RST 重置标志
};
```

**参数**

无（枚举类型）。

**返回值**

枚举值，可按位组合。

**调用（向下）**

无。

**被调用（向上）**

- `yamux::craft::handle_data()` — 检查 SYN/RST/FIN 标志
- `yamux::craft::handle_window_update()` — 检查 SYN/ACK/RST/FIN 标志
- `yamux::craft::handle_ping()` — 检查 SYN/ACK 标志

**知识域**

yamux 标志位语义、按位组合。

---

## 函数: has_flag

**功能说明**

检查标志位组合中是否包含指定标志。

**签名**

```cpp
[[nodiscard]] constexpr bool has_flag(flags f, flags flag) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `f` | `flags` | 待检查的标志组合 |
| `flag` | `flags` | 目标标志 |

**返回值**

`bool` — true 表示包含该标志。

**调用（向下）**

- `operator&(flags, flags)` — 按位与

**被调用（向上）**

- `yamux::craft::handle_data()` — 检查 flags
- `yamux::craft::handle_window_update()` — 检查 flags
- `yamux::craft::handle_ping()` — 检查 flags

**知识域**

constexpr 函数、按位操作。

---

## 枚举: go_away_code

**功能说明**

GoAway 帧的终止原因码，对应 GoAway 帧的 Length 字段。

**签名**

```cpp
enum class go_away_code : std::uint32_t {
    protocol_error = 1  // 协议错误
};
```

**参数**

无（枚举类型）。

**返回值**

枚举值，用于 GoAway 帧的 Length 字段。

**调用（向下）**

无。

**被调用（向上）**

- `yamux::craft::frame_loop()` — 大帧时发送 GoAway
- `yamux::craft::handle_go_away()` — 解析终止原因码

**知识域**

GoAway 终止原因码。

---

## 结构体: frame_header

**功能说明**

yamux 解析后的帧头（12 字节），所有多字节字段使用大端序。各字段含义因消息类型而异。

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | `uint8_t` | 协议版本（固定 0） |
| `type` | `message_type` | 消息类型 |
| `flag` | `flags` | 标志位组合 |
| `stream_id` | `uint32_t` | 流标识符（0=会话级） |
| `length` | `uint32_t` | 含义取决于消息类型 |

### frame_header::is_session

**功能说明**

检查是否为会话级消息（StreamID == 0）。

**签名**

```cpp
[[nodiscard]] bool is_session() const noexcept;
```

**参数**

无。

**返回值**

`bool` — true 表示该帧属于会话级（如 Ping、GoAway）。

**调用（向下）**

无。

**被调用（向上）**

- 测试代码 — 检查帧类型

**知识域**

会话级帧与流级帧的区分。

---

## 结构体: data_frame

**功能说明**

完整的 Data 帧（12 字节帧头 + 载荷）。用于测试和调试场景，将帧头与载荷打包返回。

| 字段 | 类型 | 说明 |
|------|------|------|
| `header` | `array<byte, 12>` | 编码后的帧头 |
| `payload` | `vector<byte>` | 帧载荷 |

---

## 函数: build_header

**功能说明**

编码帧头为 12 字节大端序数组。

**签名**

```cpp
[[nodiscard]] std::array<std::byte, frame_header_size> build_header(const frame_header &hdr) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `hdr` | `const frame_header&` | 帧头结构 |

**返回值**

`std::array<std::byte, 12>` — 编码后的 12 字节数组。

**调用（向下）**

无。

**被调用（向上）**

- `yamux::craft::push_frame()` — 编码帧头
- `yamux::build_window_update_frame()` — 构建 WindowUpdate
- `yamux::build_ping_frame()` — 构建 Ping
- `yamux::build_go_away_frame()` — 构建 GoAway
- `yamux::make_data_frame()` — 构建 Data
- `yamux::make_fin_frame()` — 构建 FIN

**知识域**

大端序编码、constexpr/noexcept 保证。

---

## 函数: parse_header

**功能说明**

解析 12 字节帧头。Version 或 Type 非法时返回 nullopt。

**签名**

```cpp
[[nodiscard]] std::optional<frame_header> parse_header(std::span<const std::byte> buffer) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `buffer` | `span<const byte>` | 输入缓冲区，至少包含 12 字节 |

**返回值**

`std::optional<frame_header>` — 解析结果，非法时返回 nullopt。

**调用（向下）**

无。

**被调用（向上）**

- `yamux::craft::frame_loop()` — 每帧读取后解析帧头

**知识域**

大端序解析、版本号/类型校验。

---

## 函数: build_window_update_frame

**功能说明**

构建 WindowUpdate 帧（仅 12 字节帧头，无载荷）。

**签名**

```cpp
[[nodiscard]] std::array<std::byte, frame_header_size> build_window_update_frame(
    flags f, std::uint32_t stream_id, std::uint32_t delta) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `f` | `flags` | 标志位，SYN 用于打开流，ACK 用于确认流 |
| `stream_id` | `uint32_t` | 流标识符 |
| `delta` | `uint32_t` | 窗口增量（字节数） |

**返回值**

`std::array<std::byte, 12>` — 编码后的 12 字节数组。

**调用（向下）**

- `yamux::build_header()` — 编码帧头

**被调用（向上）**

- 测试代码 — 构建 WindowUpdate 帧

**知识域**

WindowUpdate 帧格式。

---

## 函数: build_ping_frame

**功能说明**

构建 Ping 帧（仅 12 字节帧头，无载荷）。

**签名**

```cpp
[[nodiscard]] std::array<std::byte, frame_header_size> build_ping_frame(
    flags f, std::uint32_t ping_id) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `f` | `flags` | 标志位，SYN 为请求，ACK 为响应 |
| `ping_id` | `uint32_t` | ping 标识符 |

**返回值**

`std::array<std::byte, 12>` — 编码后的 12 字节数组。

**调用（向下）**

- `yamux::build_header()` — 编码帧头

**被调用（向上）**

- 测试代码 — 构建 Ping 帧

**知识域**

Ping 帧格式。

---

## 函数: build_go_away_frame

**功能说明**

构建 GoAway 帧（仅 12 字节帧头，无载荷）。

**签名**

```cpp
[[nodiscard]] std::array<std::byte, frame_header_size> build_go_away_frame(go_away_code code) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `code` | `go_away_code` | 终止原因码 |

**返回值**

`std::array<std::byte, 12>` — 编码后的 12 字节数组。

**调用（向下）**

- `yamux::build_header()` — 编码帧头

**被调用（向上）**

- 测试代码 — 构建 GoAway 帧

**知识域**

GoAway 帧格式。

---

## 函数: make_data_frame

**功能说明**

构建 Data 帧（帧头 + 载荷）。

**签名**

```cpp
[[nodiscard]] data_frame make_data_frame(flags f, std::uint32_t stream_id,
                                         std::span<const std::byte> payload) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `f` | `flags` | 标志位（none/SYN/FIN/RST 或其组合） |
| `stream_id` | `uint32_t` | 流标识符 |
| `payload` | `span<const byte>` | 帧载荷数据（可为空） |

**返回值**

`data_frame` — 包含帧头和载荷的结构。

**调用（向下）**

- `yamux::build_header()` — 编码帧头

**被调用（向上）**

- `yamux::make_syn_frame()` — 构建 SYN 帧
- 测试代码 — 构建 Data 帧

**知识域**

Data 帧格式。

---

## 函数: make_syn_frame

**功能说明**

构建 Data(SYN) 帧（帧头 + 载荷）。等价于 make_data_frame(flags::syn, stream_id, payload)，用于 sing-mux 兼容模式的新流创建。

**签名**

```cpp
[[nodiscard]] data_frame make_syn_frame(std::uint32_t stream_id,
                                        std::span<const std::byte> payload) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |
| `payload` | `span<const byte>` | 帧载荷数据（通常携带目标地址） |

**返回值**

`data_frame` — 包含帧头和载荷的结构。

**调用（向下）**

- `yamux::make_data_frame()` — 委托调用

**被调用（向上）**

- 测试代码 — 构建 SYN 帧

**知识域**

sing-mux 兼容模式。

---

## 函数: make_fin_frame

**功能说明**

构建 Data(FIN) 帧（仅 12 字节帧头，无载荷）。FIN 帧不携带载荷，Length 字段为 0。

**签名**

```cpp
[[nodiscard]] std::array<std::byte, frame_header_size> make_fin_frame(std::uint32_t stream_id) noexcept;
```

**参数**

| 参数 | 类型 | 说明 |
|------|------|------|
| `stream_id` | `uint32_t` | 流标识符 |

**返回值**

`std::array<std::byte, 12>` — 编码后的 12 字节数组。

**调用（向下）**

- `yamux::build_header()` — 编码帧头

**被调用（向上）**

- 测试代码 — 构建 FIN 帧

**知识域**

FIN 帧格式。
