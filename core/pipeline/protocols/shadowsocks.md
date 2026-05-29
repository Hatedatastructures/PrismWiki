---
tags: [pipeline, shadowsocks, protocol-handler, aead, sip022]
title: "SS2022 协议处理管道"
layer: core
source: "include/prism/pipeline/protocols/shadowsocks.hpp"
module: pipeline
updated: 2026-05-27
---

# SS2022 协议处理管道

Shadowsocks 2022 协议处理器。无正特征协议，通过排除法 fallback 检测。负责 AEAD 解密验证、salt 防重放、时间戳验证和双向隧道转发。

## 处理流程

1. `wrap_with_preview(use_global_mr=true)` — 全局内存池
2. 获取 thread_local salt pool（无锁防重放）
3. 创建 SS2022 relay，执行握手（AEAD 解密 → 时间戳验证 → 地址解析）
4. `relay->acknowledge()` — 乐观响应，先发 ack 再拨号
5. `forward(relay->release())` — relay 本身作为 inbound 传输层，持续 AEAD 加解密

## 设计决策

### 为什么 SS2022 使用乐观响应？

AEAD 解密成功意味着客户端合法，可以先发送 acknowledge 再拨号上游。节省一个 RTT 等待时间，对延迟敏感的连接有明显改善。

**后果**: 如果拨号上游失败，客户端已收到 ack 但连接会立即关闭。与 Mihomo 行为一致。

### 为什么 relay 本身是传输层？

SS2022 的数据传输需要持续加解密（每个 chunk 都有 AEAD 封装）。relay 继承 `transport::transmission`，`async_read_some()` 解密，`async_write_some()` 加密。上层 `tunnel()` 不感知加密细节。

**后果**: relay 的生命周期覆盖整个隧道，不能提前释放。

### 为什么 salt pool 是 thread_local？

salt 防重放检查是高频操作（每个连接），thread_local 避免锁竞争。SS2022 规范要求 salt 在一定时间窗口内唯一。

**后果**: 每个 worker 线程有独立的 salt pool，跨线程的重放攻击无法检测。但实际场景中同一客户端连接由同一 worker 处理（亲和性哈希）。

## 防重放机制

| 层级 | 机制 | 说明 |
|------|------|------|
| salt pool | thread_local set | 检测相同 salt 重放 |
| 时间戳 | `abs(now - packet_time) > tolerance` | 检测过期包重放 |
| AEAD 认证 | 解密失败即拒绝 | 篡改或错误密钥 |

## 引用关系

### 被调用

- [[core/instance/dispatch/table|dispatch]] — 注册为 SS2022 处理器

### 依赖

- [[core/pipeline/primitives|primitives]] — `wrap_with_preview`, `forward`
- [[core/crypto/aead|crypto::aead]] — AEAD 加解密
- [[core/crypto/blake3|crypto::blake3]] — BLAKE3 密钥派生
