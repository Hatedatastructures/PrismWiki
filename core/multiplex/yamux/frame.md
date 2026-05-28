---
layer: core
source: I:/code/Prism/include/prism/multiplex/yamux/frame.hpp
title: yamux::frame - yamux 协议帧格式定义
tags: [multiplex, yamux, frame, protocol, encoding]
updated: 2026-05-28
---

# yamux::frame - yamux 协议帧格式定义

## 源码位置

`I:/code/Prism/include/prism/multiplex/yamux/frame.hpp`

## 概述

定义 yamux 多路复用协议的帧格式、消息类型、标志位和编解码函数。兼容 Hashicorp/yamux 协议规范。与 smux 的 8 字节小端帧头不同，yamux 使用 12 字节大端帧头，并引入完整的流量控制和标志位系统。

## 协议常量

| 常量 | 值 | 说明 |
|------|----|------|
| `protocol_version` | 0x00 | yamux 规范规定 Version 固定为 0 |
| `frame_hdrsize` | 12 | 帧头大小（字节） |
| `default_window` | 256KB (262144) | 初始流窗口大小 |

## 帧头字节布局

12 字节定长帧头，所有多字节字段大端序（网络字节序）：

```
[Version 1B][Type 1B][Flags 2B BE][StreamID 4B BE][Length 4B BE]
```

| 偏移 | 字段 | 大小 | 说明 |
|------|------|------|------|
| 0 | Version | 1B | 固定 0x00 |
| 1 | Type | 1B | 消息类型 (0-3) |
| 2-3 | Flags | 2B BE | 可组合标志位 |
| 4-7 | StreamID | 4B BE | 流标识符，0=会话级帧 |
| 8-11 | Length | 4B BE | 含义随 Type 变化 |

## 消息类型 (message_type)

| 值 | 名称 | Length 含义 | 说明 |
|----|------|-------------|------|
| 0x00 | data | 载荷字节数 | 承载流数据或携带 SYN/FIN/RST 控制流生命周期 |
| 0x01 | window_update | 窗口增量 | 流量控制或携带 SYN/ACK 打开/确认流 |
| 0x02 | ping | ping 标识符 | SYN 为请求，ACK 为响应 |
| 0x03 | go_away | 终止原因码 | 会话终止帧 |

## 标志位 (flags)

2 字节大端序，可组合使用。

| 值 | 名称 | 说明 |
|----|------|------|
| 0x0000 | none | 无标志 |
| 0x0001 | syn | SYN 同步，打开流或发起心跳请求 |
| 0x0002 | ack | ACK 确认，确认流创建或回复心跳 |
| 0x0004 | fin | FIN 半关闭，发送端不再发送数据 |
| 0x0008 | rst | RST 重置，强制关闭流 |

### 标志位与消息类型组合语义

| 消息类型 + 标志 | 语义 |
|----------------|------|
| Data + SYN | 携带地址数据的新流创建 |
| Data + FIN | 半关闭流 |
| Data + RST | 强制重置流 |
| WindowUpdate + SYN | 打开新流（Length 为初始窗口大小） |
| WindowUpdate + ACK | 确认流创建（Length 为服务端初始窗口大小） |
| Ping + SYN | 心跳请求 |
| Ping + ACK | 心跳响应 |

## GoAway 原因码 (away_code)

| 值 | 名称 | 说明 |
|----|------|------|
| 1 | protocol_error | 收到无法识别的帧或非法状态转换 |

## 接口表

### frame_header 结构

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| version | uint8_t | protocol_version | 协议版本，固定为 0 |
| type | message_type | data | 消息类型 |
| flag | flags | none | 标志位组合 |
| stream_id | uint32_t | 0 | 流标识符，会话级帧为 0 |
| length | uint32_t | 0 | 长度字段，含义取决于消息类型 |

方法：`is_session()` 检查是否为会话级消息（StreamID == 0）。

### data_frame 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| header | array\<byte, 12\> | 编码后的帧头 |
| payload | memory::vector\<byte\> | 帧载荷 |

### 编解码函数

| 函数 | 输入 | 返回 | 说明 |
|------|------|------|------|
| `build_header` | frame_header | array\<byte, 12\> | 编码帧头为大端序数组 |
| `parse_header` | span\<const byte\> | optional\<frame_header\> | 解析 12 字节帧头，Version/Type 非法时返回 nullopt |
| `build_winupd` | flags, stream_id, delta | array\<byte, 12\> | 构建 WindowUpdate 帧 |
| `build_ping` | flags, ping_id | array\<byte, 12\> | 构建 Ping 帧 |
| `build_goaway` | away_code | array\<byte, 12\> | 构建 GoAway 帧 |
| `build_data` | flags, stream_id, payload | data_frame | 构建 Data 帧 |
| `build_syn` | stream_id, payload | data_frame | 构建 Data(SYN) 帧，等价于 build_data(flags::syn, ...) |
| `build_fin` | stream_id | array\<byte, 12\> | 构建 Data(FIN) 帧，无载荷 |

### 辅助函数

| 函数 | 说明 |
|------|------|
| `operator&(flags, flags)` | 标志位按位与运算 |
| `has_flag(flags, flags) -> bool` | 检查标志位组合中是否包含指定标志 |

## 关联文档

- [[core/multiplex/yamux/craft|yamux::craft]] - yamux 协议实现
- [[core/multiplex/yamux/config|yamux::config]] - yamux 协议配置
