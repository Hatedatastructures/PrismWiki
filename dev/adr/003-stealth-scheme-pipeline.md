---
title: "ADR 003: 伪装方案管道架构"
created: 2026-05-23
updated: 2026-05-23
layer: dev
tags: [dev, adr, architecture, stealth, pipeline]
status: accepted
---

# ADR 003: 伪装方案管道架构

## 状态

已接受 (Accepted)

## 背景

Prism 需要在单个监听端口上同时支持多种 TLS 伪装方案（Reality、ShadowTLS、Restls、AnyTLS、TrustTunnel、Native TLS）。这带来了以下挑战：

- **多方案共存**: 不同伪装方案使用不同的认证机制（X25519 密钥交换、HMAC-SHA1、应用层认证），需要在不执行握手的前提下区分客户端使用的是哪种方案
- **成本差异**: 有些检测成本极低（字节比较），有些需要昂贵的密码学运算（HMAC 验证、AEAD 解密）
- **确定性差异**: Reality 有独占特征（session_id 标记），而 Restls/TrustTunnel 只能通过 SNI 模糊匹配
- **可扩展性**: 未来可能新增伪装方案（如 ECH、AnyTLS 变体），架构需要支持插拔式扩展
- **降级安全**: 非法连接（主动探测、误连）需要安全回退，不应暴露服务端指纹

### 方案 A: 硬编码顺序检测

按固定顺序依次尝试每种方案。

- **优势**: 实现简单，易于理解
- **劣势**: 不可扩展；高成本检测可能在低成本检测前执行；方案间有互斥关系时处理困难

### 方案 B: 策略模式 + 工厂

每种方案实现统一接口，通过工厂创建。

- **优势**: 面向对象设计，可扩展
- **劣势**: 缺乏检测的分层优化，所有方案等同对待

### 方案 C: Tier 分层 + 方案管道

按检测成本和确定性分层，构建管道式的检测流程。每层有明确的职责和终止条件。

- **优势**: 高效（低成本检测优先，命中即停）；可扩展（新方案只需声明 Tier）；确定性保证
- **劣势**: 架构复杂，分层逻辑需要仔细设计

## 决策

**选择方案 C: Tier 分层 + 方案管道**

### 分层检测架构

```
SNI 路由表 (scheme_route_table)
    |
    | 根据 ClientHello SNI 筛选候选方案
    v
Tier 0: sniff() -- 零成本字节比较
    |   Reality: session_id[0:3] == [0x01, 0x08, 0x02]
    |   命中后设 solo=true，跳过后续所有检测
    v
Tier 1: verify() -- 有成本验证
    |   ShadowTLS: HMAC-SHA1 验证
    |   AnyTLS: 应用层认证
    |   命中后设 solo_flag != 0，跳过 Tier 2
    v
Tier 2: guess() -- 模糊匹配
    |   Restls: SNI 路由匹配
    |   TrustTunnel: SNI 路由匹配
    |   Native: 兜底方案
    |   按权重分排序
    v
handshake() -- 执行握手
    | 按候选列表顺序依次执行
    | 成功则返回结果
    | 失败则尝试下一个
    | 全部失败则 fallback 到真实 TLS
    v
结果: { transport, detected, preread, error }
```

### 方案基类设计

```cpp
class stealth_scheme {
public:
    virtual auto name() const -> std::string_view = 0;
    virtual auto tier() const -> uint8_t { return 2; }      // 默认 Tier 2
    virtual auto unique() const -> bool { return false; }    // 是否独占
    virtual auto active(const config &cfg) const -> bool = 0;

    // Tier 0: 零成本字节比较
    virtual auto sniff(uint32_t bitmap, const hello_features &f) const -> sniff_result;

    // Tier 1: 有成本验证
    virtual auto verify(const hello_features &f, span<const byte> raw, const config &cfg) const -> verify_result;

    // Tier 2: 模糊匹配
    virtual auto guess(const config &cfg) const -> verify_result;

    // 执行握手
    virtual auto handshake(handshake_context ctx) -> awaitable<handshake_result> = 0;
};
```

### 方案注册表

单例模式的方案注册表，启动时注册所有方案：

```cpp
stealth::register_all_schemes();  // main() 中调用
```

新方案只需：
1. 实现 `stealth_scheme` 子类
2. 在 `register_all_schemes()` 中添加注册调用
3. 添加对应的配置结构体

### SNI 路由表

通过 ClientHello 中的 SNI 字段进行初步筛选，将候选方案范围缩小到 SNI 匹配的子集。每种伪装方案配置独立的 SNI 白名单（`server_names`），只有 SNI 匹配的方案才进入检测管道。

### 评分机制

- **Tier 0**: 命中即确定（`solo = true`），直接返回
- **Tier 1**: 评分制（0-1000），`solo_flag != 0` 表示确定性命中
- **Tier 2**: 权重分制，按权重排序作为候选列表

### 实际方案实现对照

| 方案 | tier() | unique() | sniff() | verify() | guess() |
|------|--------|----------|---------|----------|---------|
| Reality | 0 | true | session_id 标记 | - | - |
| ShadowTLS | 1 | true | - | HMAC-SHA1 | - |
| AnyTLS | 1 | false | - | 应用层认证 | - |
| Restls | 2 | false | - | - | SNI 匹配 |
| TrustTunnel | 2 | false | - | - | SNI 匹配 |
| Native | 2 | false | - | - | 兜底 |

### 检测流程示例

以一个 Reality + ShadowTLS + Restls 共存的配置为例：

```
ClientHello 到达:
  1. SNI 路由: www.microsoft.com -> Reality, www.apple.com -> ShadowTLS
  2. Tier 0 sniff():
     - Reality: 检查 session_id[0:3] == [0x01, 0x08, 0x02]
     - 命中 -> solo=true, 直接返回 Reality 候选
  3. (如果 Tier 0 未命中) Tier 1 verify():
     - ShadowTLS: HMAC-SHA1 验证 SessionID
     - 命中 -> solo_flag != 0, 返回 ShadowTLS 候选
  4. (如果 Tier 1 未命中) Tier 2 guess():
     - Restls: SNI 匹配 -> 按权重排序
     - Native: 兜底
  5. 按候选列表依次执行 handshake()
```

> 参考: [[core/stealth/overview|伪装方案总览]]、[[core/recognition/layered_pipeline|识别管道]]

## 后果

### 优势

1. **高效检测**:
   - Reality（Tier 0）只需 3 字节比较即可确定性判定，零成本
   - ShadowTLS/AnyTLS（Tier 1）仅在 Tier 0 未命中时才执行
   - Restls/TrustTunnel（Tier 2）作为兜底，仅在所有确定性检测失败时执行
   - 平均检测延迟远低于顺序检测方案

2. **可扩展性**:
   - 新方案只需实现 `stealth_scheme` 接口并声明 `tier()` 级别
   - 管道自动将新方案插入正确的检测层级
   - 不影响已有方案的检测逻辑

3. **确定性保证**:
   - Tier 0 方案有独占特征（如 Reality 的 session_id 标记），命中后跳过所有后续检测
   - 避免了多方案之间的误判
   - 主动探测（无有效特征）安全回退到真实 TLS

4. **优雅降级**:
   - 所有伪装方案失败后，fallback 到 Native TLS（使用自签或自有证书）
   - 主动探测者看到的是真实 TLS 握手（Reality/ShadowTLS）或标准 HTTPS 服务
   - 不暴露代理服务的存在

5. **配置驱动**:
   - 方案通过配置启用/禁用（`active(const config&)`）
   - SNI 白名单支持精细化的路由控制
   - 新方案无需修改核心检测逻辑

### 劣势

1. **架构复杂度**:
   - 三层检测 + 评分机制 + SNI 路由的组合逻辑较复杂
   - 新开发者需要理解 Tier 分层、评分机制、独占标记等概念
   - 调试时需要追踪多层检测的执行路径

2. **Tier 分配需要判断力**:
   - 新方案的 Tier 分配需要根据其检测特征的成本和确定性来判断
   - Tier 分配不当可能导致误判或性能损失
   - 例如：如果将本该 Tier 0 的方案分配到 Tier 2，会增加不必要的检测开销

3. **评分机制可能不够精确**:
   - Tier 2 的权重分是静态配置的，不能根据运行时信息动态调整
   - 多个 Tier 2 方案 SNI 重叠时，评分可能无法准确区分
   - 极端情况下可能需要尝试多个候选方案

4. **SNI 路由的限制**:
   - 依赖 ClientHello 中的 SNI 字段，某些客户端可能不发送 SNI
   - ECH 加密 SNI 后，SNI 路由将失效，需要依赖 ECH 解密后的信息
   - SNI 白名单配置需要与伪装方案一一对应

### 缓解措施

- 详尽的日志记录每层检测的结果和原因
- 单元测试覆盖所有 Tier 分层的组合场景
- 通过 [[docs/protocol-matrix|协议能力矩阵]] 文档化每种方案的检测层级
- 在 [[dev/extending/overview|扩展开发]] 中提供新方案接入指南

## 相关页面

- [[core/stealth/overview|伪装方案总览]]
- [[core/recognition/layered_pipeline|识别管道]]
- [[docs/protocol-matrix|协议能力矩阵]]
- [[dev/adr/004-protocol-recognition|ADR 004: 协议智能识别]]
- [[dev/extending/overview|扩展开发]]
