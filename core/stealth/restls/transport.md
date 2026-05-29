---
layer: core
source: include/prism/stealth/restls/transport.hpp
title: Restls Transport
created: 2026-05-25
updated: 2026-05-27
tags: [stealth, restls, transport, auth-mac, mask, transmission]
---
# restls_transport

> 源码位置: include/prism/stealth/restls/transport.hpp + src/prism/stealth/restls/transport.cpp

## 设计决策（WHY）

### 为什么 read/write 使用 auth_mac + mask 双层验证

每帧的读取和写入都需要两步密码学操作：先计算 auth_mac（8 字节 BLAKE3 keyed hash）验证帧完整性，再用 mask（4 字节）XOR 混淆 payload 的长度和命令字段。双层设计将认证和混淆分离：auth_mac 确保数据不被篡改，mask 确保流量特征不可识别。

### 为什么 auth_mac 输入包含 counter

counter 是递增的帧序号（big-endian 8 字节），包含在 auth_mac 的输入中。这防止了重放攻击——即使攻击者捕获了一帧并重新发送，counter 不匹配会导致 auth_mac 验证失败。

### 为什么 mask 同时混淆长度和命令

Restls 帧的明文结构中，长度和命令字段是流量分析的主要目标。通过 XOR mask 混淆这两个字段，被动探测者无法从加密记录中识别帧的真实长度分布和命令类型。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| auth_mac 固定 8 字节 | appdata_maclen | 截断到前 8 字节 |
| mask 固定 4 字节 | mask_len | XOR 混淆密钥 |
| counter 每帧递增 | 防重放 | 不能重用 |
| max_plaintext = 16384 | 帧大小限制 | 超过拒绝 |

## 失败场景

| 场景 | 触发条件 | 后果 |
|------|----------|------|
| auth_mac 验证失败 | 数据被篡改或密码错误 | 连接终止 |
| counter 不匹配 | 帧重放或丢失 | auth_mac 验证失败 |
| 帧超过 max_plaintext | 数据过大 | 拒绝处理 |
| BLAKE3 计算失败 | 密钥无效 | 未定义行为 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| transport -> crypto | 调用 | compute_auth_mac / compute_mask |
| transport -> handshake | 被创建 | 接收 handshake_detail |
| transport -> channel::transport::transmission | 继承 | 实现 async_read_some / async_write_some |
