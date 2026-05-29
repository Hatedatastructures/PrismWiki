---
layer: core
source: include/prism/multiplex/smux/frame.hpp
title: smux::frame - smux 帧协议定义
tags: [multiplex, smux, frame, protocol, encoding]
updated: 2026-05-28
---

# smux::frame - smux 帧协议定义

## 源码位置

`include/prism/multiplex/smux/frame.hpp`

## 概述

定义 smux 多路复用协议的帧格式、命令类型和编解码函数。兼容 Mihomo/xtaci/smux v1。smux 采用 8 字节定长帧头，Length 和 StreamID 使用小端字节序。

## 协议常量

| 常量 | 值 | 说明 |
|------|----|------|
| `protocol_version` | 0x01 | 协议版本号 |
| `frame_hdrsize` | 8 | 帧头大小（字节） |
| `max_frame_length` | 65535 | 最大帧数据大小（64KB） |

## 帧头字节布局

8 字节定长帧头，Length 和 StreamID 使用小端字节序：

```
[Version 1B][Cmd 1B][Length 2B LE][StreamID 4B LE]
```

## 命令类型 (command)

| 值 | 名称 | 说明 |
|----|------|------|
| 0 | syn | 新建流 |
| 1 | fin | 半关闭流 |
| 2 | push | 数据推送 |
| 3 | nop | 心跳（不回复） |

## frame_header 结构

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| version | uint8_t | protocol_version | 协议版本号 |
| cmd | command | push | 命令类型 |
| length | uint16_t | 0 | 负载长度（小端序） |
| stream_id | uint32_t | 0 | 流标识符（小端序），0 表示会话级帧 |

## 地址解析结构

### parsed_address

从 mux 首个 PSH 帧解析的目标地址。sing-mux StreamRequest 格式：`[Flags 2B][ATYP 1B][Addr(var)][Port 2B]`。

Flags 含义：bit0 = UDP 流标识，bit1 = PacketAddr 模式。

| 字段 | 类型 | 说明 |
|------|------|------|
| host | memory::string | 目标主机 |
| port | uint16_t | 目标端口 |
| offset | size_t | 地址结束位置（相对于原始 buffer） |
| is_udp | bool | 是否为 UDP 流（Flags bit0） |
| addr | addr_mode | 地址编码模式（Flags bit1） |

### udp_dgram

UDP 数据报解析结果，格式：`[ATYP 1B][Addr(var)][Port 2B][Data]`。

| 字段 | 类型 | 说明 |
|------|------|------|
| host | memory::string | 目标主机 |
| port | uint16_t | 目标端口 |
| payload | span\<const byte\> | 数据部分（不含 UDP 头部） |
| consumed | size_t | 解析消耗的总字节数 |

### udp_prefixed

Length-prefixed UDP 数据报，格式：`[Length 2B BE][Payload]`。目标地址在 SYN 时已确定，不包含在数据帧中。

| 字段 | 类型 | 说明 |
|------|------|------|
| payload | span\<const byte\> | 数据部分 |
| consumed | size_t | 解析消耗的总字节数 |

## 接口表

### 解析函数

| 函数 | 输入 | 返回 | 说明 |
|------|------|------|------|
| `parse_dgram` | span\<const byte\>, mr | optional\<udp_dgram\> | 解析 UDP 数据报（SOCKS5 地址格式） |
| `parse_prefixed` | span\<const byte\> | optional\<udp_prefixed\> | 解析 length-prefixed UDP 数据报 |
| `parse_address` | span\<const byte\>, mr | optional\<parsed_address\> | 解析 mux 首个 PSH 中的 Flags+地址 |
| `deserialization` | span\<const byte\> | optional\<frame_header\> | 解析帧头（至少 8 字节） |

### 构建函数

| 函数 | 输入 | 返回 | 说明 |
|------|------|------|------|
| `build_dgram` | datagram_params, mr | memory::vector\<byte\> | 构建 UDP 数据报 |
| `build_prefixed` | span\<const byte\>, mr | memory::vector\<byte\> | 构建 length-prefixed UDP 数据报 |

### datagram_params 结构

| 字段 | 类型 | 说明 |
|------|------|------|
| host | string_view | 目标主机 |
| port | uint16_t | 目标端口 |
| payload | span\<const byte\> | 数据负载 |

## 地址类型支持

| ATYP | 地址类型 | 格式 |
|------|----------|------|
| 0x01 | IPv4 | 4 字节 |
| 0x03 | 域名 | [Length 1B][Name] |
| 0x04 | IPv6 | 16 字节 |

## 关联文档

- [[core/multiplex/smux/craft|smux::craft]] - smux 协议实现
- [[core/multiplex/parcel|parcel]] - UDP 数据报管道（使用解析函数）
