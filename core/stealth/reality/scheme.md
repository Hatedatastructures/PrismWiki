---
layer: core
source: I:/code/Prism/include/prism/stealth/reality/scheme.hpp
title: Reality Scheme
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

继承 [[stealth-scheme]] 基类，实现 Reality 协议的检测和握手。

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

调用 [[config]] 的 `enabled()` 方法判断配置完整性。

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

调用 [[handshake]] 执行 Reality 认证流程。

#### 处理流程

1. 检查 session 上下文有效性
2. 调用 [[handshake]] 函数
3. 根据结果类型处理：

| 结果类型 | 处理 |
|----------|------|
| `authenticated` | 返回加密传输层 (seal)，检测协议为 VLESS |
| `not_reality` | 返回原始传输层，检测协议为 TLS |
| `fallback` | 回退完成，透明代理已执行 |
| `failed` | 根据错误类型处理（TLS 记录错误转 TLS，其他返回错误） |

## 调用链

- [[stealth-scheme]] ← 继承基类
- [[config]] ← 配置结构体
- [[handshake]] ← 执行握手
- [[seal]] ← 加密传输层