---
tags: [recognition, confidence]
layer: core
module: recognition
source: include/prism/recognition/confidence.hpp
title: confidence.hpp
---

# confidence.hpp

检测置信度枚举定义，指导方案执行顺序。

## 核心类型

### confidence

```cpp
enum class confidence : std::uint8_t
{
    high,   // 高置信度：特征完全匹配
    medium, // 中置信度：特征部分匹配
    low,    // 低置信度：不确定
    none    // 无特征：标准 TLS
};
```

| 值 | 说明 | 执行策略 |
|----|------|----------|
| `high` | 特征完全匹配，可直接执行 | 直接执行，无需验证，跳过 L2/L3 |
| `medium` | 特征部分匹配，需完整验证 | 优先执行，失败后继续候选 |
| `low` | 特征部分匹配但不确定 | 靠后执行，需验证 |
| `none` | 标准 TLS，无伪装特征 | 执行 Native 兜底方案 |

## 设计决策

### 为什么是四级置信度而非二元判断？

伪装方案的检测不是精确的——同一 ClientHello 可能匹配多个方案的特征。四级置信度（high/medium/low/none）允许分层检测管道在候选方案之间排序。`high` 触发确定性命中（短路），`medium`/`low` 进入候选列表按序尝试，`none` 直接走 Native TLS。二元判断（是/否）无法表达"可能是 Reality 但也可能是 ShadowTLS"的模糊情况。

**后果**: 执行时间与置信度成反比——`high` 最快（< 0.1ms），`low` 最慢（2-5ms）。

### 确定性命中短路

当任一方案达到 `confidence::high` 时，检测管道立即短路返回，跳过昂贵的 L2/L3 层计算（HMAC 验证、熵分析）。

**后果**: 高置信度场景下检测延迟 < 0.1ms。但如果高置信度方案握手失败（如密钥不匹配），不会回退到其他候选。

## 约束

### 置信度阈值不可动态修改

**类型**: 资源上限

**规则**: 阈值（0.85/0.50/0.20）是编译期常量。无法在运行时调整。

**违反后果**: 不适用。

**源码依据**: `confidence.hpp` 枚举定义 + `pipeline` 中的映射逻辑

## 使用场景

| 场景 | 置信度 | 触发条件 |
|------|--------|----------|
| Reality 标记命中 | `high` | session_id 前缀 `[01:08:02]` |
| ShadowTLS HMAC 通过 | `high` | HMAC 验证成功 |
| SNI 匹配但无其他特征 | `medium` | SNI 在路由表中 |
| 仅 session_id 异常 | `low` | 长度/熵异常 |
| 标准 TLS 浏览器 | `none` | 无任何伪装特征 |

## 引用关系

### 被引用

- [[core/recognition/result|result]]：分析结果结构使用
- [[core/recognition/layered_pipeline|layered_pipeline]]：分层检测管道使用
- [[core/stealth/scheme|stealth::scheme]]：方案检测返回值