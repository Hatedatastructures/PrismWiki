---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/scheme.hpp
title: Reality Scheme
tags:
  - stealth
  - reality
  - tier0
  - exclusive
  - scheme
---

# Reality Scheme

Reality 伪装方案类，封装 Reality TLS 握手和协议检测逻辑。

## 源码位置

- 头文件：`I:/code/Prism/include/prism/stealth/reality/scheme.hpp`
- 实现文件：`I:/code/Prism/src/prism/stealth/reality/scheme.cpp`

## 类定义

```cpp
class scheme final : public stealth_scheme
```

继承 [[core/stealth/scheme|stealth_scheme]] 基类，实现 Reality 协议的检测和握手。

## 基本信息

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `name()` | `"reality"` | 方案名称 |
| `tier()` | `0` | Tier 0 方案（快速检测） |
| `unique()` | `true` | 独占方案，命中后跳过其他方案 |
| `weight()` | `450` | 权重值 |

## 配置检查

### active

检查 Reality 配置是否启用。

```cpp
auto active(const psm::config &cfg) const noexcept -> bool
```

调用 [[core/stealth/reality/config|config]] 的 `enabled()` 方法判断配置完整性。

### snis

返回允许的 SNI 列表。

```cpp
auto snis(const psm::config &cfg) const -> memory::vector<memory::string>
```

## Tier 0 快速检测

### sniff

```cpp
auto sniff(std::uint32_t bitmap,
           const protocol::tls::client_hello_features &features) const
    -> sniff_result
```

检测逻辑（按置信度排序）：

### 检测规则

| 条件 | 置信度 (hint) | 独占 (solo) | 说明 |
|------|---------------|-------------|------|
| `reality_marker_01_08_02` | 950 | true | session_id 前缀 `[01:08:02]` |
| `has_x25519 + has_full_session_id` | 450 | false | 有 X25519 公钥且 session_id=32 |
| `has_x25519 + session_id_non_standard` | 400 | false | 有 X25519 且非标准 session_id |
| `has_x25519` | 200 | false | 仅有 X25519 |
| `has_sni + has_full_session_id` | 100 | false | 有 SNI 且 session_id=32 |
| `has_sni` | 100 | false | 仅有 SNI |

### 独占标记

Reality 使用 session_id 前缀作为独占特征：

```
session_id[0] == 0x01
session_id[1] == 0x08
session_id[2] == 0x02
```

命中此标记立即返回 `solo=true`，跳过其他方案。

## 执行握手

### handshake

```cpp
auto handshake(stealth::handshake_context ctx)
    -> net::awaitable<stealth::handshake_result>
```

调用 [[core/stealth/reality/handshake|handshake]] 执行 Reality 认证流程。

#### 处理流程

1. 检查 session 上下文有效性
2. 调用 [[core/stealth/reality/handshake|handshake]] 函数
3. 根据结果类型处理：

| 结果类型 | 处理 |
|----------|------|
| `authenticated` | 返回加密传输层 (seal)，检测协议为 VLESS |
| `not_reality` | 返回原始传输层，检测协议为 TLS |
| `fallback` | 回退完成，透明代理已执行 |
| `failed` | 根据错误类型处理（TLS 记录错误转 TLS，其他返回错误） |

## 设计决策（WHY）

### 为什么 sniff 是 5 级评分而非二元判断

Reality 客户端并非都带有 `[01:08:02]` 标记。不同版本的 Reality 客户端实现可能有不同的 session_id 编码方式。5 级评分（950/450/400/200/100）允许执行器在没有独占标记时仍然将 Reality 纳入候选。最高级 950 配合 `solo=true` 确保有标记时独占执行；低级别允许 Reality 与其他方案竞争，由后续 `verify()` 或 `handshake()` 最终判定。

### 为什么 `weight()` 返回 450

虽然 Reality 是 Tier 0 方案不使用 `guess()`（因此 `weight()` 不参与检测），450 这个值是 Reality 独占标记的 hint 值附近，保持了命名一致性。实际上 Tier 0 的 `sniff()` 完全绕过了 `guess()`，`weight()` 在这里仅作为元数据。

### 为什么 `handshake()` 需要 `session` 而非仅 `config`

Reality 的 fallback 逻辑需要通过 `session->worker.router` 连接目标服务器。这是所有伪装方案中唯一需要路由器的——因为 Reality 可能需要将非 Reality 客户端的流量透明代理到 `dest` 服务器。没有 `session` 则无法执行 fallback，直接返回 `not_supported`。

### 为什么 `verify()`/`guess()` 不被重写

Reality 的身份确认完全在 `handshake()` 内部完成（X25519 密钥交换 + short_id 验证 + HKDF 认证密钥派生 + AES-GCM 解密 session_id）。这个流程需要完整的 ClientHello 原始字节和网络 I/O，不适合放在 `verify()`（无 I/O）或 `guess()`（无特征分析）中。

## 约束

| 约束 | 来源 | 说明 |
|------|------|------|
| `unique() == true` | 方案定义 | 独占标记 `[01:08:02]` 命中后不再检测其他方案 |
| `tier() == 0` | 方案定义 | 最高优先级，零成本检测 |
| `session` 必须非空 | `handshake()` | 无 session 无法执行 fallback |
| `sniff()` 不做 I/O | Tier 0 语义 | 只检查 bitmap 中的特征位 |
| 认证客户端不获取 dest 证书 | 握手注释 | 使用合成 Ed25519 证书 |

## 失败场景

| 场景 | 触发条件 | 行为 |
|------|----------|------|
| 独占命中但认证失败 | `[01:08:02]` 标记 + short_id 错误 | 执行器无回退，连接失败 |
| 无标记 + SNI 不匹配 | 低评分候选但 SNI 不在白名单 | 跳过 Reality，尝试其他方案 |
| `session` 为空 | 测试或异常调用路径 | 返回 `not_supported` |
| fallback 到 dest 失败 | dest 服务器不可达 | 返回 `reality_dest_unreachable` |

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| `scheme` → `config` | 依赖 | `active()` 调用 `config::enabled()`，`snis()` 从 config 提取 SNI |
| `scheme` → `recognition::tls` | 依赖 | `sniff()` 使用 `rec_tls::has_feature()` 检查 bitmap |
| `scheme` → `reality::handshake` | 调用 | `handshake()` 委托给 reality::handshake 函数 |
| `scheme` → `session_context` | 依赖 | 需要 `server_ctx.config()` 和 `worker.router` |

## 调用链

- [[core/stealth/scheme|stealth-scheme]] ← 继承基类
- [[core/stealth/reality/config|config]] ← 配置结构体
- [[core/stealth/reality/handshake|handshake]] ← 执行握手
- [[core/stealth/reality/seal|seal]] ← 加密传输层