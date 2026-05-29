---
tags: [recognition, result]
layer: core
module: recognition
source: include/prism/recognition/result.hpp
title: result.hpp
---

# result.hpp

ClientHello 特征分析结果结构定义。

## 核心类型

### analysis_result

ClientHello 特征分析结果，由各 scheme 的 `detect()` 返回。

| 字段 | 类型 | 说明 |
|------|------|------|
| `candidates` | `memory::vector<memory::string>` | 候选方案名称列表（按置信度排序，high 在前，none 不加入） |
| `score` | [[core/recognition/confidence\|confidence]] | 整体置信度，取最高候选的置信度，无候选时为 `none` |
| `features` | `hello_features` | 提取的 ClientHello 原始特征 |
| `error` | `fault::code` | 解析错误码，成功时为 `success` |

## 约束

### candidates 排序保证

**类型**: 状态前置

**规则**: `candidates` 按置信度降序排列。调用方可按顺序依次尝试执行，首个成功即可停止。

**违反后果**: 高成本方案优先执行，增加延迟。

**源码依据**: `result.hpp:29`

### score 与 candidates 一致性

**类型**: 状态前置

**规则**: `score` 必须是 `candidates` 中最高置信度方案的级别。空 `candidates` 时 `score` 为 `none`。

**违反后果**: 执行策略决策错误（如 `high` 但 `candidates` 为空）。

**源码依据**: `result.hpp:33`

## 引用关系

### 依赖

- [[core/recognition/confidence|confidence]]：置信度枚举
- [[core/protocol/tls/types|protocol::tls::hello_features]]：TLS 特征结构
- [[core/fault/code|fault::code]]：错误码

### 被引用

- [[core/recognition/recognition|recognition]]：识别入口使用
- [[core/recognition/layered_pipeline|layered_pipeline]]：分层检测管道使用
- [[core/stealth/scheme|stealth::scheme]]：方案检测返回值