---
tags: [transformer, overview]
layer: core
module: transformer
source: I:/code/Prism/include/prism/transformer/json.hpp
title: Transformer 模块
---

# Transformer 模块

Transformer 模块提供数据序列化与反序列化功能，基于 glaze 库实现 JSON 格式支持。

## 设计决策

### 为什么选 glaze 而非 nlohmann/json？

glaze 是编译期反射 JSON 库，通过 `glz::meta` 特化直接映射结构体字段，无需运行时 JSON DOM。序列化/反序列化是编译期生成的代码，零堆分配（除输出字符串外）。nlohmann/json 构建运行时 DOM 树，每个 JSON 节点都是一次堆分配。

**后果**: 需要为每个序列化类型定义 `glz::meta` 特化或 `glz::meta<T>` 自动推导。不支持动态 JSON 结构。

### 为什么 deserialize 先构造 temp 再 move？

glaze 反序列化可能中途失败。如果直接写入 `value`，失败时 `value` 处于部分修改的无效状态。先反序列化到 `temp`，成功后再 `std::move` 到 `value`，保证 `value` 要么完整要么不变。

**后果**: 要求目标类型可默认构造且可移动赋值。不满足的类型直接写入 `value`（有部分修改风险）。

## 模块组成

| 组件 | 说明 | 源码 |
|------|------|------|
| [[core/transformer/json]] | JSON序列化 | `prism/transformer/json.hpp` |

## 核心接口

| 函数 | 说明 |
|------|------|
| `serialize(value, out)` | 序列化为 `memory::string`，返回 bool |
| `serialize(value, out, ec)` | 同上，带错误上下文 |
| `serialize(value, mr)` | 返回新 `memory::string` |
| `deserialize(json, value)` | 反序列化，返回 bool |
| `deserialize(json, value, ec)` | 同上，带错误上下文 |
| `deserialize(json, value, parse_opts)` | 带 PMR 内存资源 |
| `deserialize(json, value, mr)` | 带 PMR，忽略错误 |

## 约束

### 大 JSON 消耗内存

**类型**: 资源上限

**规则**: 反序列化大 JSON 文档时，glaze 内部缓冲区和目标对象都会消耗内存。

**违反后果**: 配置文件过大可能导致 OOM。

**源码依据**: `json.hpp:9`

## 引用关系

### 被引用

- [[core/loader/load|loader]]：配置加载使用 JSON 反序列化