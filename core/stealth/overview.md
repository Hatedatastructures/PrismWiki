---
layer: core
source: include/prism/stealth/
title: Stealth 模块总览
tags:
  - stealth
  - scheme
  - tier-detection
  - TLS-camouflage
  - core-module
---

# Stealth 模块总览

> 源码位置: `include/prism/stealth/`

## 模块职责

Stealth 是 TLS 伪装方案模块，负责：

- **方案抽象**: 定义 `stealth_scheme` 基类，统一伪装方案接口
- **分层检测**: Tier 0/1/2 三级检测架构，从快速到详细逐层筛选
- **方案执行**: 按优先级依次尝试方案，直到匹配成功
- **方案注册**: 单例注册表管理所有伪装方案
- **Native 兜底**: 标准 TLS 方案作为最终兜底

## 文件结构

### 头文件

```
include/prism/stealth/
├── scheme.hpp           # 方案基类定义
├── executor.hpp         # 方案执行器
├── registry.hpp         # 方案注册表
├── native.hpp           # Native 兜底方案
├── anytls/              # AnyTLS 伪装方案
├── ech/                 # ECH 伪装方案
├── reality/             # Reality 伪装方案
├── restls/              # Restls 伪装方案
├── shadowtls/           # ShadowTLS 伪装方案
└── trusttunnel/         # TrustTunnel 伪装方案
```

## 核心组件

| 组件 | 头文件 | 职责 |
|------|--------|------|
| [[scheme|方案基类]] | `scheme.hpp` | 抽象基类、检测结果、执行上下文 |
| [[executor|执行器]] | `executor.hpp` | 方案管道执行、候选筛选 |
| [[registry|注册表]] | `registry.hpp` | 单例方案注册、查询 |
| [[native|Native方案]] | `native.hpp` | 标准 TLS 兜底方案 |

## 分层检测架构

Stealth 模块采用三级检测架构，按成本从低到高逐层筛选：

```
入站 TLS 连接
       │
       ▼
┌─────────────────────────────────────┐
│  Tier 0: sniff() 零成本检测          │
│  - 字节比较，无 HMAC/解密            │
│  - 如 Reality session_id 标记       │
│  - 命中后可独占跳过其他方案           │
└─────────────────────────────────────┘
       │ 未命中
       ▼
┌─────────────────────────────────────┐
│  Tier 1: verify() 有成本验证         │
│  - HMAC 验证或解密                   │
│  - 如 ShadowTLS HMAC、AnyTLS ECH    │
│  - 返回评分 (0-1000)                 │
└─────────────────────────────────────┘
       │ 未命中
       ▼
┌─────────────────────────────────────┐
│  Tier 2: guess() 模糊匹配            │
│  - 依赖 SNI 路由                     │
│  - 如 Restls、TrustTunnel、Native   │
│  - 作为兜底方案                      │
└─────────────────────────────────────┘
```

## 执行流程

```
recognition 分析 ClientHello
       │
       ├── 特征位图提取
       ├── Tier 0 快速检测
       ├── Tier 1 详细检测
       ├── Tier 2 模糊匹配
       │
       ▼
生成候选方案列表
       │
       ▼
scheme_executor 执行
       │
       ├── 按候选顺序尝试
       ├── 每个方案执行 handshake()
       │       ├── 返回 TLS: 不是此方案，继续下一个
       │       └── 返回具体协议: 匹配成功
       │
       ├── 全部失败 → Native 兜底
       │
       ▼
返回 handshake_result
       ├── transport: 最终传输层
       ├── detected: 检测到的内层协议
       └── preread: 预读数据
```

## 方案列表

| 方案 | Tier | 独占 | 检测方式 | 说明 |
|------|------|------|----------|------|
| Reality | 0 | 是 | session_id 标记 | 最早期的 TLS 伪装方案 |
| ShadowTLS | 1 | 是 | HMAC 验证 | 真实 TLS 握手伪装 |
| AnyTLS | 1 | 是 | ECH 解密 | 加密 ClientHello |
| Restls | 2 | 否 | SNI 匹配 | 模糊匹配方案 |
| TrustTunnel | 2 | 否 | SNI 匹配 | 模糊匹配方案 |
| Native | 2 | 否 | 兜底 | 标准 TLS 处理 |

## 故障模式

### 空壳方案

以下伪装方案当前为框架状态，`handshake()` 直接返回 `detected=tls`：
- **Restls** — Tier 2，score=100
- **AnyTLS** — Tier 2，支持 ECH 扩展检测但解密未实现
- **TrustTunnel** — Tier 2，score=100

配置这些方案不会生效，流量走 native 兜底（标准 TLS 终止）。

### 方案选择与故障传播

- 确定性命中（Tier 0/1 独占）→ 单方案执行，失败无回退
- 候选方案执行失败 + 可 rewind → 尝试下一方案
- 候选方案执行失败 + 不可 rewind → 终止或 native 兜底

详见 [[dev/debugging/deep-dive/stealth-limitations|伪装方案执行器限制与故障分析]]

## 设计决策（WHY）

### 为什么采用三级检测架构

代理伪装方案面临一个核心矛盾：**检测成本与准确性的权衡**。Tier 0/1/2 架构的分层不是随意的——每一级对应一种"确定性程度"：

- **Tier 0 (sniff)**：字节级特征可以直接断言"一定是"或"一定不是"。Reality 的 `session_id[0:3]` 标记 `[0x01, 0x08, 0x02]` 是协议硬编码的，不存在误判空间。零成本的代价是只有极少数方案能提供这种特征。
- **Tier 1 (verify)**：HMAC/解密验证后可以高置信度确认身份，但需要计算。ShadowTLS 的 SessionID HMAC、AnyTLS 的 ECH 解密都属此类。因为成本高，安排在 Tier 0 之后执行。
- **Tier 2 (guess)**：仅凭 SNI 匹配，无法确定连接是否属于该方案。多个方案可能共享同一 SNI。因此 Tier 2 使用评分制 + 执行时 try-rewind，允许多方案竞争。

这种分层确保了：**每次连接最多只执行一个有成本的验证操作**，其余方案要么零成本排除，要么延迟到执行阶段才判断。

### 为什么需要 `polluted` 标记

`handshake_result.polluted` 是一个关键安全阀。一旦方案向连接写入了数据（如 Reality 发送 ServerHello），传输层状态就无法回滚到写入前。如果此时方案最终失败，rewind 会丢弃已发送的数据，但客户端已经接收了这些字节——这种不一致会导致后续所有方案的握手必然失败。`polluted=true` 告知执行器跳过 rewind，直接终止管道或走 native 兜底。

### 为什么注册顺序等于优先级

`register_schemes()` 的调用顺序直接决定 `scheme_registry::all()` 的返回顺序。这不是巧合——执行器 `execute_by_analysis` 在无候选列表时按注册顺序逐个尝试，因此**注册顺序就是 fallback 顺序**。这意味着 Reality 必须第一个注册（Tier 0 独占检测），Native 必须最后注册（兜底）。

## 约束

| 约束 | 来源 | 影响 |
|------|------|------|
| 注册时机：启动阶段一次性 | `register_schemes()` 在 `main()` 中调用 | 运行时 `scheme_registry` 不可变，新增/删除方案需重启 |
| `snis()` 必须与配置的 SNI 精确匹配 | 各方案 `snis()` 从 config 提取 | SNI 配置错误直接导致 Tier 2 方案不可达 |
| `handshake()` 不可阻塞 | 协程纯度要求 | 所有 I/O 必须异步，禁止 `sleep_for`/阻塞 read |
| `weight()` 仅 Tier 2 有效 | 基类默认 `guess()` 使用 `weight()` 作为 score | Tier 0/1 方案的 `weight()` 无实际作用 |
| `unique()==true` 的方案可以独占跳过后续 | 执行器检测到 `solo` 后立即终止 | 独占方案失败无回退，必须极其可靠 |

## 失败场景

| 场景 | 触发条件 | 表现 | 后果 |
|------|----------|------|------|
| Tier 0 独占误判 | Reality `sniff()` 返回 `hit=true` 但 ClientHello 不完整 | 执行 Reality `handshake()`，失败无回退 | 连接中断 |
| rewind 失败 | 方案写入数据后失败（`polluted=true`） | `try_rewind()` 返回 false | 管道终止或走 native |
| 空候选列表 | 所有 Tier 0/1/2 都未命中 | 按注册顺序执行全部方案 | 性能退化但不断连 |
| 配置冲突 | 多个 Tier 2 方案共享同一 SNI | SNI 无法区分，按 weight 排序尝试 | 可能执行错误方案 |
| Native 也失败 | TLS 握手失败（证书错误等） | 连接中断 | 无法恢复 |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `recognition` → `stealth` | 调用 | recognition 解析 ClientHello 后，将 `analysis_result`（含 candidates）传递给 `scheme_executor::execute_by_analysis()` |
| `stealth` → `transport` | 依赖 | 所有方案的 `handshake_context.inbound` 是 `shared_transmission`，方案可能用 `snapshot` 包装后 rewind |
| `stealth` → `protocol` | 依赖 | `handshake_result.detected` 类型为 `protocol::protocol_type`，决定后续分发的协议处理器 |
| `stealth` → `config` | 依赖 | `active()`、`snis()`、`verify()` 都读取 `psm::config`，配置格式变更会影响所有方案 |
| `stealth` → `connect` | 依赖 | 部分 `handshake()` 需要连接后端服务器（Reality fallback、ShadowTLS relay） |
| `agent` ← `stealth` | 被依赖 | session 的 `diversion()` 使用 `handshake_result` 决定分发路径 |

## 变更敏感性

| 变更类型 | 影响范围 | 风险 |
|----------|----------|------|
| 新增 `stealth_scheme` 子类 | 需在 `register_schemes()` 中注册，否则不会被检测到 | 低：注册即可用 |
| 修改 `sniff_result`/`verify_result` 结构 | 所有方案的检测逻辑和执行器的判断逻辑 | 高：接口变更 |
| 修改 `handshake_result.polluted` 语义 | `execute_pipeline` 的 rewind 策略 | 高：可能导致 rewind 失效 |
| 变更 `register_schemes()` 注册顺序 | 无候选时的 fallback 顺序 | 中：可能导致错误的兜底方案 |
| 修改 `recognition::analysis_result.candidates` 格式 | `execute_by_analysis` 的候选解析 | 高：候选列表为空时的行为完全不同 |

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[core/channel/overview|Channel]] | 传输层抽象 |
| 依赖 | [[core/protocol/overview|Protocol]] | 协议类型、TLS 特征 |
| 依赖 | [[core/recognition/overview|Recognition]] | ClientHello 分析结果 |
| 依赖 | [[core/fault/overview|Fault]] | 错误码 |
| 依赖 | [[core/memory/overview|Memory]] | PMR 容器 |
| 被依赖 | [[core/instance/overview|Instance]] | 会话处理使用伪装方案 |

## 相关文档

- [[scheme|方案基类详解]] — stealth_scheme 抽象类定义
- [[executor|执行器详解]] — scheme_executor 管道执行
- [[registry|注册表详解]] — scheme_registry 单例管理
- [[native|Native 方案]] — 标准 TLS 兜底方案
- [[core/recognition/recognition|Recognition 模块]] — 协议识别与候选生成