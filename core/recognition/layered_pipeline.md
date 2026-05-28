---
tags: [recognition, layered_pipeline]
layer: core
module: recognition
source: I:/code/Prism/include/prism/recognition/pipeline.hpp
title: layered_detection_pipeline
---

# layered_detection_pipeline

分层检测管道，按成本和确定性分层执行检测。

## 设计理念

避免不必要的计算，按成本分层：

| 层级 | 成本 | 检测方式 | 示例 |
|------|------|----------|------|
| Tier 0 | 零成本 | 字节比较 | Reality session_id 标记 |
| Tier 1 | 有成本 | HMAC/解密验证 | ShadowTLS HMAC |
| Tier 2 | 模糊 | SNI 路由匹配 | SNI 匹配配置 |

### 为什么分三层而非一层全部检测？

性能关键。Tier 0（字节比较）耗时 < 0.01ms，Tier 1（HMAC）约 0.1ms，Tier 2（SNI 哈希查找）< 0.01ms。但若同时执行所有层，即使 Tier 0 已确定性命中也会浪费 Tier 1/2 的计算。分层短路设计确保大多数连接在 Tier 0 即完成识别。

**后果**: Tier 0 的独占标记（如 Reality `[01:08:02]`）一旦命中，后续所有检测被跳过。如果标记存在但密钥不匹配，连接直接失败，不会尝试其他方案。

## 核心类型

### candidate_entry

| 字段 | 说明 |
|------|------|
| `name` | 方案名称 |
| `score` | 评分（0-1000，越高越确定） |
| `tier` | 检测层级 (0-2) |
| `is_deterministic` | 是否确定性命中（独占特征） |

### pipeline_result

| 字段 | 说明 |
|------|------|
| `deterministic_hit` | 是否确定性命中（独占特征） |
| `exclusive_scheme` | 独占命中的方案名称 |
| `candidates` | 候选列表（按评分排序） |
| `reason` | 检测原因（日志用） |

### detect_input

将 detect 方法的多个输入参数收敛为单一结构（项目规范：函数参数 <= 3）。

| 字段 | 说明 |
|------|------|
| `bitmap` | 特征位图 |
| `features` | ClientHello 特征结构（引用） |
| `raw` | 原始 ClientHello 字节 |
| `cfg` | 全局配置（引用） |

## layered_detection_pipeline

### 构造

```cpp
explicit layered_detection_pipeline(
    const std::vector<stealth::shared_scheme> &schemes);
```

从 stealth 注册表构建管道，内部将方案按 `tier()` 返回值分配到三层。

### detect()

```cpp
auto detect(detect_input input,
            const std::vector<stealth::shared_scheme> &matched_schemes) const
    -> pipeline_result;
```

### 检测流程

```
Tier 0 → 独占特征命中？→ 直接返回单一候选
    │ 否
    ▼
Tier 1 → HMAC验证通过？→ 返回单一候选
    │ 否
    ▼
Tier 2 → SNI路由匹配 → 返回多候选列表
    │ 兜底
    ▼
Native TLS
```

## 约束

### detect_input 引用生命周期

**类型**: 生命周期

**规则**: `detect_input` 中的 `features` 和 `cfg` 为 `const` 引用，必须在 `detect()` 返回前有效。

**违反后果**: 悬挂引用，读取无效数据。

**源码依据**: `pipeline.hpp:91-97`

### 方案注册必须在 pipeline 构造前完成

**类型**: 调用顺序

**规则**: `stealth::register_schemes()` 必须在 `layered_detection_pipeline` 构造前调用（`main.cpp` 启动流程保证）。

**违反后果**: pipeline 中无方案，所有 TLS 流量走 Native 兜底。

**源码依据**: `main.cpp` 启动顺序

## 引用关系

### 依赖

- [[core/recognition/confidence|confidence]]：置信度枚举
- [[core/recognition/scheme-route-table|scheme-route-table]]：SNI 路由表
- [[core/stealth/scheme|stealth::scheme]]：伪装方案
- [[core/protocol/tls/types|protocol::tls::hello_features]]：TLS 特征

### 被引用

- [[core/recognition/recognition|recognition]]：identify() 中使用