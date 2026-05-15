---
title: "result.hpp — 识别模块分析结果结构定义"
source: "include/prism/recognition/result.hpp"
module: "recognition"
type: api
tags: [recognition, result, 分析结果, 结构体]
created: 2026-05-15
updated: 2026-05-15
related:
  - recognition/recognition
  - recognition/confidence
  - recognition/layered_pipeline
  - protocol/tls/types
  - fault/code
  - stealth/executor
---

# result.hpp

> 源码: `include/prism/recognition/result.hpp`
> 模块: [[recognition|Recognition]]

## 概述

定义 ClientHello 分析结果结构，由各 scheme 的 `detect()` 返回。该结果不包含网络 I/O 相关数据，仅描述特征分析的结论。`analysis_result` 包含按置信度排序的候选方案列表，供方案执行器按顺序尝试。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container\|container]] | PMR 容器类型 |
| 依赖 | [[recognition/confidence\|confidence]] | 置信度枚举 |
| 依赖 | [[protocol/tls/types\|types]] | ClientHello 特征结构 |
| 依赖 | [[fault/code\|code]] | 错误码 |
| 被依赖 | [[recognition/recognition\|recognition]] | identify 流程使用分析结果 |
| 被依赖 | [[stealth/executor\|executor]] | execute_by_analysis 使用候选列表 |

## 命名空间

`psm::recognition`

---

## 结构体: analysis_result

### 功能说明

ClientHello 特征分析结果。由 `registry::analyze()` 返回，包含按置信度排序的候选方案列表。调用方应根据 `candidates` 的顺序依次尝试方案执行。`score` 字段反映最高置信度方案的级别，为 `none` 时表示无任何匹配。

### 签名

```cpp
struct analysis_result
{
    memory::vector<memory::string> candidates;
    confidence score{confidence::none};
    protocol::tls::client_hello_features features;
    fault::code error{fault::code::success};
};
```

### 参数表格

无（结构体定义）。

### 返回值

无（结构体定义）。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::vector<memory::string>` | `candidates` | 候选方案名称列表（按置信度排序，high 在前，none 不加入） |
| `confidence` | `score` | 整体置信度，取最高候选的置信度值，无候选时为 `none` |
| `protocol::tls::client_hello_features` | `features` | 提取的 ClientHello 原始特征（SNI、session_id、key_share 等） |
| `fault::code` | `error` | 解析错误码，成功时为 `fault::code::success` |

### 调用（向下）

无（纯数据结构）。

### 被调用（向上）

| 调用方 | 模块 | 说明 |
|--------|------|------|
| `identify()` | [[recognition/recognition\|recognition]] | 构建 analysis_result 传入 executor |
| `scheme_executor::execute_by_analysis()` | [[stealth/executor\|executor]] | 按 candidates 顺序执行方案 |

### 知识域

- [[recognition/confidence\|置信度枚举]]
- [[protocol/tls/types\|TLS 类型]]
- [[fault/code\|错误码]]

---

## 数据流

```
scheme::detect() → analysis_result{candidates, score, features, error}
                          │
                          ▼
identify() → executor.execute_by_analysis(analysis_result, handshake_context)
                          │
                          ▼
                  按 candidates 顺序尝试方案执行
```

## 知识域

- [[ref/anti-censorship/tls-fingerprint\|TLS 指纹分析]]
- [[recognition/confidence\|置信度]]
- [[stealth/executor\|方案执行器]]
- [[protocol/tls/types\|TLS 类型]]
