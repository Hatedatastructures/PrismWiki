---
title: 协议识别弱点与误判分析
source:
  - I:/code/Prism/include/prism/recognition/probe/analyzer.hpp
  - I:/code/Prism/src/prism/recognition/recognition.cpp
module: debugging
type: deep-dive
tags: [debugging, recognition, misclassification, probe, fingerprint, failure-analysis]
layer: dev
created: 2026-05-21
updated: 2026-05-21
related:
  - "[[core/recognition/overview]]"
  - "[[dev/debugging/protocol]]"
---

# 协议识别弱点与误判分析

## 概述

Prism 的协议识别机制采用三阶段流水线架构：

```
Probe(24B 前缀) → Identify(ClientHello 解析) → Execute(scheme 执行)
```

每个阶段都基于特征匹配（fingerprint matching）做决策，而特征匹配本质上是启发式的——
它依赖字节模式的统计特性，而非密码学强度的确定性判断。这意味着在所有可能的
输入空间中，**始终存在误判窗口**。

| 阶段 | 输入 | 决策依据 | 失败后果 |
|------|------|----------|----------|
| **Probe** | 首包前 24 字节 | 首字节/固定偏移的特征值 | 协议归类错误 |
| **Identify** | TLS ClientHello | SNI 提取 + 候选匹配 | 无候选或候选错误 |
| **Execute** | 完整连接上下文 | scheme 执行 + rewind 兜底 | 连接失败 |

本文档系统分析每个阶段的弱点、误判概率及对应的排障策略。

## Probe 阶段弱点

### 检测顺序与排除法

Probe 阶段按照固定优先级对首包的前 24 字节进行模式匹配，检测顺序为：

```
SOCKS5(0x05) → TLS(0x16 0x03) → HTTP(方法名 ASCII) → Shadowsocks(排除法)
```

前三个协议有明确的正特征，而 Shadowsocks 没有正特征——它是通过排除法归类的。
这意味着**任何不匹配前三者的流量都会被当作 Shadowsocks**。

### SS2022 salt 误判为 TLS

SS2022 的请求头以随机 salt 开头。salt 是完全随机的字节序列，因此其首字节有
1/256 的概率恰好是 `0x16`（TLS ClientHello 的固定首字节）。

代码通过二级检查来缓解此问题——验证第二字节是否为 `0x03`（TLS 版本高位）：
将误判概率从 1/256 降低到 1/65536。

**但问题并未完全消除。** 当 SS2022 salt 的前两字节恰好是 `0x16 0x03` 时，
流量仍会被误判为 TLS。

| 指标 | 值 |
|------|-----|
| 误判概率 | 1/65536（约 0.0015%） |
| 误判后果 | SS2022 连接被当作 TLS → ClientHello 解析失败 → 可能走 native 兜底或直接失败 |
| 实际触发频率 | 高并发场景下每 65536 个连接可能出现一次 |

**诊断方法**：如果 SS2022 客户端偶尔连接失败，且日志中出现 TLS 解析错误
（例如 `ClientHello parse error` 或 `invalid handshake type`），
应首先排查此误判。

### VLESS 仅 4 字节弱特征

VLESS 协议在 Probe 阶段的检测条件为：

```
b0 == 0x00
b17 == 0x00
b18 ∈ {0x01, 0x02, 0x7F}
b21 ∈ {0x01, 0x02, 0x03}
```

仅约束了 4 个字节的位置，且条件相对宽松。相比 TLS 的 `0x16 0x03` 两字节
固定匹配或 SOCKS5 的 `0x05` 单字节匹配，VLESS 的区分度明显更低。

计算随机数据命中 VLESS 特征的理论概率：

```
P(b0=0) × P(b17=0) × P(b18∈{1,2,0x7F}) × P(b21∈{1,2,3})
= 1/256 × 1/256 × 3/256 × 3/256
≈ 1/4,722,366,482,930,000 (极低)
```

理论概率极低，但在实际场景中：

- VLESS 只在 TLS 内层（识别阶段之后）被检测，Probe 阶段的外层流量
  不会被误判为 VLESS
- 如果 VLESS 被直接暴露在 TCP 面上（非标准部署），理论上存在被其他
  随机流量"淹没"的风险，但实际发生概率可以忽略

### 排除法归类问题

排除法是 Probe 阶段最大的系统性弱点：

```
不匹配 SOCKS5(1B) → 不匹配 TLS(2B) → 不匹配 HTTP(4B+) → 归为 Shadowsocks
```

**影响场景**：

- 部署了新的非标准协议，流量会被错误归类为 Shadowsocks
- 某些协议的握手数据恰好不匹配任何已知特征（例如基于 UDP 的协议、
  自定义二进制协议）
- 网络噪音或攻击流量也会被当作 Shadowsocks 处理

**间接影响**：Shadowsocks 处理逻辑会尝试解密这些流量，产生大量
`crypto_error` 或 `invalid_salt` 错误日志，干扰正常排障。

## Identify 阶段弱点

### SNI 路由依赖 ClientHello

Identify 阶段的核心逻辑是解析 TLS ClientHello 并提取 SNI（Server Name Indication），
然后用 SNI 匹配候选 scheme 列表。

**脆弱点**：

1. **ClientHello 截断**：如果首包长度不足，ClientHello 可能不完整，
   SNI extension 可能缺失。此时无法提取域名，路由失败。
2. **格式异常**：某些客户端或中间设备可能修改 ClientHello 格式
   （例如 GREASE 扩展、非标准扩展顺序），导致解析器跳过 SNI。
3. **无 SNI**：客户端不发送 SNI（例如直接使用 IP 地址连接），
   Identify 无法匹配任何候选。

**兜底行为**：无 SNI 时，系统会使用默认候选列表或 native 兜底。
这意味着如果一个 scheme 只配置了 SNI 匹配规则而未设置默认匹配，
该 scheme 将永远不会被选中。

### 分层检测的确定性

Identify 阶段采用分层（Tier）匹配策略，不同层有不同的确定性级别：

| 层级 | 协议 | 匹配机制 | 确定性 |
|------|------|----------|--------|
| Tier 0 | Reality | session_id 格式 | 结构性匹配，确定性高 |
| Tier 1 | ShadowTLS | HMAC 验证 | 密码学匹配，确定性高 |
| Tier 2 | 其他 | SNI + 模糊匹配 | 启发式匹配，确定性中等 |

**Tier 0 Reality 独占标记**：只有特定格式的 `session_id` 才会触发 Reality
识别。这是一个结构性的特征——Reality 的 session_id 包含编码后的目标信息，
其长度和格式与正常 TLS 不同。

**Tier 1 ShadowTLS HMAC**：需要正确的密码才能通过 HMAC 验证。
密码学保证使得误判概率可以忽略。

**Tier 2 模糊匹配**：当 Tier 0 和 Tier 1 都未命中时，进入 Tier 2 的
模糊匹配。此阶段依赖 SNI 字符串匹配和协议特征推断，确定性相对较低。

**层级穿透问题**：如果 Tier 0 的 Reality 检测条件过于宽松，
可能将非 Reality 流量误判为 Reality，导致后续处理失败。
反之，如果条件过于严格，可能遗漏部分 Reality 流量。

## 识别失败传播链

识别失败不会立即导致连接断开，而是沿着预设的兜底路径传播：

```
Probe失败
  → 归类为默认协议（通常是 Shadowsocks）
  → 走 Shadowsocks 处理逻辑
  → 解密失败 → 连接断开

Probe归类为TLS → Identify无候选
  → Native 兜底（标准 TLS 代理行为）
  → 可能成功（如果目标确实是普通 TLS 服务）
  → 或失败（如果目标是代理协议需要特殊处理）

Probe归类为TLS → 候选执行全失败
  → 尝试 rewind（回退到连接初始状态）
  → rewind 成功 → 尝试下一个候选
  → 全部不可 rewind → Native 兜底

Probe归类为非TLS（错误）
  → 走错误协议的处理逻辑
  → 连接必然失败
  → 日志中可能出现协议解析错误
```

关键观察：**Probe 错误的分类无法被后续阶段纠正。** Identify 和 Execute
都建立在 Probe 的分类结果之上。如果 Probe 将流量错误归类为非 TLS，
后续阶段不会尝试 TLS 解析。

## 误判场景排障指南

### 常见误判模式与诊断

| 症状 | 可能的误判 | 排查方法 |
|------|-----------|----------|
| SS2022 偶尔连接失败 | salt 首字节 `0x16 0x03` 误判为 TLS | 检查日志中是否有 TLS 解析错误与 SS2022 连接时间重合 |
| VLESS 连接被当作其他协议 | 特征匹配冲突 | 检查识别日志中的 Probe result 和特征匹配详情 |
| 未知协议连接走 SS 处理 | 排除法归类 | 检查 Probe 日志，确认流量是否被归为 Shadowsocks |
| Reality 连接失败 | session_id 格式不匹配 Tier 0 | 检查 ClientHello 解析日志，验证 session_id 格式 |
| 配置了多个 scheme 但只有部分生效 | SNI 匹配规则冲突 | 检查 Identify 日志中的候选列表和 SNI 匹配结果 |

### 关键日志标签

排障时应关注以下日志标签：

```
[Recognition] Probe result: type=<协议类型> confidence=<置信度>
[Recognition] Starting identify lifecycle for type=<协议类型>
[Recognition] SNI '<domain>' matched N schemes
[Recognition] Tier <N> matched: <scheme_name>
[Recognition] No candidates found, falling back to native
[Recognition] All candidates failed, attempting rewind
```

### 缓解策略

1. **SS2022 salt 误判**：增加检测字节数，例如检查 salt 的更多字节
   是否符合 TLS ClientHello 的版本号分布。但需权衡——检查越多字节，
   延迟越高。
2. **排除法归类**：考虑引入"未知协议"类型，而非默认归为 Shadowsocks。
   对于未知流量，直接拒绝连接或记录告警，避免产生大量解密错误。
3. **SNI 缺失**：为无法提取 SNI 的场景配置默认 scheme 匹配规则，
   避免完全依赖 SNI 路由。
4. **监控告警**：对 `crypto_error` 和 `invalid_salt` 错误率设置监控，
   突然增加可能指示 Probe 误判导致流量被错误归类。
