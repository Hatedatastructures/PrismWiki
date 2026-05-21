---
title: 伪装方案执行器限制与故障分析
source:
  - I:/code/Prism/src/prism/stealth/executor.cpp
  - I:/code/Prism/src/prism/recognition/recognition.cpp
module: debugging
type: deep-dive
tags: [debugging, stealth, executor, rewind, snapshot, failure-analysis]
layer: dev
created: 2026-05-21
updated: 2026-05-21
related:
  - "[[core/stealth/executor]]"
  - "[[core/stealth/scheme]]"
  - "[[core/recognition/overview]]"
---

# 伪装方案执行器限制与故障分析

## 概述

Scheme Executor 是伪装方案的调度管道，负责按优先级逐个尝试已注册的伪装方案，直到某个方案握手成功或全部失败后回退到 native 兜底。

方案注册顺序（即默认优先级）：

1. **Reality**
2. **ShadowTLS**
3. **RestLS**
4. **AnyTLS**
5. **TrustTunnel**
6. **Native**（兜底，始终注册在末尾）

Executor 的核心循环逻辑：对候选列表中的每个方案依次调用 `handshake()`，成功则返回结果；失败时检查是否可以 `rewind`（回卷连接上的读写状态），可以则 rewind 后尝试下一个方案，不可以则直接终止。

---

## Rewind 不可逆限制

### Snapshot 机制

Executor 在开始调度前调用 `ensure_snapshot()`，将 inbound 连接包装为一层 snapshot。Snapshot 的作用是记录当前连接的读写状态，在方案失败时尝试回卷到调度前的干净状态。

- `ensure_snapshot()` — 如果 inbound 尚未被 snapshot 包装，则创建 snapshot 并替换 inbound 指针
- `can_rewind()` — 检查 snapshot 是否还处于可回卷状态（即 `wrote_ == false`）
- `try_rewind()` — 先检查 `can_rewind()`，满足条件后调用底层 `rewind()` 将连接状态重置

### 关键约束

一旦任何方案向 inbound 写入数据，snapshot 内部的 `wrote_` 标志被设为 `true`，此后 `can_rewind()` 永远返回 `false`，后续方案无法再 rewind。

这意味着：**第一个向连接写入数据的方案决定了 rewind 的命运。如果该方案后续失败，整个调度链将无法回退。**

### Reality 的不可逆点

Reality 握手过程中，`async_write_scatter` 发送 ServerHello 是第一个写操作。一旦 ServerHello 发出：

- snapshot 的 `wrote_` 变为 `true`
- 如果 Reality 在 Stage 4（消费客户端 Finished）失败，传输层已被 ServerHello 污染
- 后续方案看到的是不完整的 TLS 数据流，而不是原始的 ClientHello
- rewind 无法撤回已经发送到对端的字节

### ShadowTLS 的不可逆点

ShadowTLS 将客户端的 ClientHello 转发到后端 TLS 服务器：

- 后端可能已经开始发送响应数据（ServerHello、Certificate 等）
- 这些数据已经通过 socket 发出，rewind 只能重置本地缓冲区，无法撤回对端已接收的字节
- 如果 ShadowTLS 握手在后续阶段失败，rewind 的意义有限

---

## 方案选择机制

Scheme Executor 并非盲目尝试所有方案，而是根据 Probe 阶段收集的信息对方案进行分层匹配：

### Tier 0（零成本）

Reality 独占标记（`reality_marker_01_08_02`）位于 ClientHello 的特定位置。如果检测到该标记：

- 确定性命中 Reality 方案
- 候选列表中只保留 Reality
- 只执行单一方案

### Tier 1（有成本验证）

ShadowTLS 通过 HMAC 校验 ClientHello 中的认证字段：

- 确定性命中 ShadowTLS 方案
- 候选列表中只保留 ShadowTLS
- 只执行单一方案

### Tier 2（模糊匹配）

以下方案无法通过确定性标记识别，走评分排序：

| 方案 | 默认评分 | 说明 |
|---|---|---|
| RestLS | 100 | TLS 流量模式匹配 |
| AnyTLS | 100 | 检测 ECH 扩展 |
| TrustTunnel | 100 | 特征匹配 |
| Native | 50 | 兜底方案 |

按评分从高到低依次尝试，同分时按注册顺序。

### 确定性命中的风险

Tier 0 和 Tier 1 命中后，Executor 只执行单一方案。如果该方案握手失败，Executor **直接返回错误**，不会尝试其他方案。

这是一个设计权衡：确定性标记意味着协议协商已完成，客户端应该只说一种协议。如果握手失败，说明客户端实现有 bug 或连接被篡改，尝试其他方案没有意义。

---

## Native 兜底

### 触发条件

- 所有候选方案均已尝试且全部失败
- 或者候选列表为空

### 执行逻辑

1. 执行标准 TLS 握手（使用配置的证书）
2. 握手完成后读取最多 128 字节，探测内层协议
3. 如果读取了 60+ 字节仍无法识别具体协议，通过排除法返回 `shadowsocks`
4. 整个过程本质上是一个标准 TLS 终止，没有任何伪装

### 意味着

当 Native 兜底被触发时，连接降级为普通的 TLS 服务器。客户端的流量特征（SNI、ALPN 等）会以明文形式暴露在 TLS 握手阶段，容易受到主动探测的识别。

---

## 空壳方案

以下方案的 `handshake()` 实现直接返回 `detected=tls`，不执行任何实际的伪装操作：

### RestLS

- Tier 2，score=100
- 不执行任何握手逻辑，直接标记为 TLS 协议
- 实际上是一个占位实现

### AnyTLS

- Tier 2，score=100
- 检测 ClientHello 中的 ECH（Encrypted Client Hello）扩展
- 但 ECH 解密功能尚未实现（`decrypt.cpp` 检查格式后直接返回 `not_supported`）
- 检测到 ECH 后仍然返回 `detected=tls`，没有后续处理

### TrustTunnel

- Tier 2，score=100
- 同样是空壳实现

### ECH 解密

`decrypt.cpp` 中的流程：

1. 检查 ClientHello 是否包含 ECH 扩展
2. 验证 ECH 扩展的格式是否正确
3. 返回 `not_supported`，不做任何解密操作

### 实际影响

用户在配置中声明使用这些方案，但实际上这些方案不执行任何伪装逻辑。流量最终会走 native 兜底，表现为标准的 TLS 终止。这在排查问题时容易造成困惑：日志显示方案被选中，但行为与未配置伪装一致。

---

## 故障传播链

```
Probe(24B) → detect(SOCKS5/TLS/HTTP/SS)
  → TLS → Identify(ClientHello) → SNI路由 → 分层检测
    → Tier0命中 → 执行单方案(如Reality) → 失败 → 返回错误(无回退)
    → Tier1命中 → 执行单方案(如ShadowTLS) → 失败 → 返回错误
    → Tier2多候选 → 按评分依次执行
      → A失败 + 可rewind → rewind → 尝试B
      → A失败 + 不可rewind → pass_through或终止
      → 全部失败 → native兜底
  → 非TLS → 直接处理(不经过SchemeExecutor)
```

### 各阶段失败的行为

| 阶段 | 失败行为 | 是否可恢复 |
|---|---|---|
| Probe 阶段 | 返回 `unknown`，连接被丢弃 | 否 |
| Tier 0 命中后握手失败 | 返回错误，连接关闭 | 否 |
| Tier 1 命中后握手失败 | 返回错误，连接关闭 | 否 |
| Tier 2 方案 A 失败（未写入） | rewind 后尝试方案 B | 是 |
| Tier 2 方案 A 失败（已写入） | 无法 rewind，连接终止 | 否 |
| 全部 Tier 2 失败 | native 兜底 | 是（降级） |

---

## 排障方法

### 日志标签

Scheme Executor 的所有日志统一使用 `[SchemeExecutor]` 前缀。

### 关键日志消息

| 日志消息 | 含义 |
|---|---|
| `Scheme 'X' succeeded` | 方案 X 握手成功，连接正常建立 |
| `Scheme 'X' failed but snapshot rewound` | 方案 X 失败但成功 rewind，将尝试下一个候选 |
| `Scheme 'X' failed with error: ...` | 方案 X 失败且无法 rewind，连接终止 |
| `All candidates failed, executing native fallback` | 所有方案失败，进入 native 兜底 |

### 常见排障场景

**确认方案是否生效**

检查日志中是否出现 `Scheme 'X' succeeded`。如果只有候选选中日志但没有 succeeded，说明方案握手失败了。

**确认是否走了 native 兜底**

搜索 `native fallback` 日志。如果出现，说明所有伪装方案均失败或为空壳，连接降级为标准 TLS。

**Reality 握手在 Stage 4 失败**

这是典型的不可逆失败。日志会显示 `failed with error`，没有 rewind 日志。此时应检查：
- 客户端时间与服务器时间偏差是否过大（Reality 使用时间验证）
- 客户端的 public key 是否与服务器配置匹配
- 网络中间设备是否修改了 ClientHello

**ShadowTLS 握手失败**

检查 HMAC 密钥配置是否一致。ShadowTLS 依赖共享密钥验证 ClientHello，密钥不匹配会导致 Tier 1 命中后握手失败，且不会回退到其他方案。

**空壳方案误导**

如果日志显示 RestLS/AnyTLS/TrustTunnel 被选中并 succeeded，但实际上连接行为与标准 TLS 一致，说明命中了空壳实现。此时应关注 `detected` 字段：空壳方案返回 `detected=tls`，而非具体的伪装协议。
