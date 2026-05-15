---
title: "Transformer 模块"
module: "transformer"
type: module
tags: [transformer, json, glaze, 序列化, 反序列化]
created: 2026-05-15
updated: 2026-05-15
related:
  - memory/container
  - loader/load
  - agent/config
  - stealth/registry
---

# Transformer 模块

> 模块: [[transformer|transformer]]

## 概述

Transformer 模块负责 JSON 序列化/反序列化，基于 glaze 库实现。提供高性能的编译期反射 JSON 解析，集成 PMR 内存系统，支持自定义内存分配器。所有配置结构体通过 glaze 的编译期反射实现自动序列化。

---

## transformer.hpp — JSON 序列化聚合头文件

> 源码: `include/prism/transformer.hpp`
> 模块: [[transformer|transformer]]

### 概述

Transformer 模块的公共聚合头文件，引入 `transformer/json.hpp`。作为模块入口，外部代码只需包含此文件即可使用全部 JSON 序列化功能。

### 命名空间

`psm::transformer`

---

## json.hpp — glaze JSON 适配

> 源码: `include/prism/transformer/json.hpp`
> 模块: [[transformer|transformer]]

### 概述

glaze JSON 库的适配层，提供类型安全的 JSON 解析。所有函数均为模板函数，通过 glaze 的编译期反射自动处理序列化/反序列化。支持移动语义优化：对于支持移动构造的类型，使用临时对象避免破坏原对象状态。

### 命名空间

`psm::transformer::json`

### 函数模板: serialize() [基础版]

- **功能说明**: 将对象序列化为 JSON 字符串。序列化前清空输出字符串，失败时也会清空。
- **签名**: `template <typename StructureObject> [[nodiscard]] bool serialize(const StructureObject &value, memory::string &out)`
- **参数**: `value` — 待序列化的对象实例；`out` — 用于存储序列化结果的字符串（使用前会被清空）
- **返回值**: `bool` — true 序列化成功，false 序列化失败
- **调用（向下）**: `glz::write_json()`
- **被调用（向上）**: `serialize(value, out, out_ec)`、`serialize(value, mr)`
- **知识域**: JSON 序列化

### 函数模板: serialize() [带错误上下文]

- **功能说明**: 将对象序列化为 JSON 字符串，同时输出错误上下文。适用于需要诊断序列化失败原因的场景。
- **签名**: `template <typename StructureObject> [[nodiscard]] bool serialize(const StructureObject &value, memory::string &out, glz::error_ctx &out_ec)`
- **参数**: `value` — 待序列化的对象实例；`out` — 用于存储序列化结果的字符串；`out_ec` — 用于存储序列化错误上下文
- **返回值**: `bool` — true 序列化成功，false 序列化失败
- **调用（向下）**: `glz::write_json()`
- **被调用（向上）**: 需要错误上下文的序列化场景
- **知识域**: JSON 序列化（带诊断）

### 函数模板: serialize() [带内存资源]

- **功能说明**: 将对象序列化为 JSON 字符串，使用指定的内存资源分配输出缓冲区。
- **签名**: `template <typename StructureObject> [[nodiscard]] memory::string serialize(const StructureObject &value, memory::resource_pointer mr = memory::current_resource())`
- **参数**: `value` — 待序列化的对象实例；`mr` — 内存资源指针（默认使用当前线程的内存资源）
- **返回值**: `memory::string` — 序列化后的 JSON 字符串，失败时为空字符串
- **调用（向下）**: `serialize(value, out)`、`memory::current_resource()`
- **被调用（向上）**: 需要直接返回 JSON 字符串的场景
- **知识域**: JSON 序列化（PMR 分配）

### 函数模板: deserialize() [基础版]

- **功能说明**: 将 JSON 字符串反序列化为对象。对于支持移动构造的类型，使用临时对象解析后再移动赋值，避免破坏原对象状态（原子语义）。
- **签名**: `template <typename StructureObject> [[nodiscard]] bool deserialize(std::string_view json_data, StructureObject &value)`
- **参数**: `json_data` — JSON 原始数据；`value` — 接收数据的对象引用
- **返回值**: `bool` — true 反序列化成功，false 反序列化失败
- **调用（向下）**: `glz::read_json()`
- **被调用（向上）**: `loader::load()` 配置文件解析
- **知识域**: JSON 反序列化

### 函数模板: deserialize() [带错误上下文]

- **功能说明**: 将 JSON 字符串反序列化为对象，同时输出错误上下文。
- **签名**: `template <typename StructureObject> [[nodiscard]] bool deserialize(std::string_view json_data, StructureObject &value, glz::error_ctx &out_ec)`
- **参数**: `json_data` — JSON 原始数据；`value` — 接收数据的对象引用；`out_ec` — 用于存储反序列化错误上下文
- **返回值**: `bool` — true 反序列化成功，false 反序列化失败
- **调用（向下）**: `glz::read_json()`
- **被调用（向上）**: `deserialize(json_data, value, mr, out_ec)`
- **知识域**: JSON 反序列化（带诊断）

### 函数模板: deserialize() [带内存资源 + 错误上下文]

- **功能说明**: 将 JSON 字符串反序列化为对象，使用指定的内存资源构造临时对象。对于接受 `memory::resource_pointer` 构造参数的类型，临时对象会使用指定的内存资源。
- **签名**: `template <typename StructureObject> [[nodiscard]] bool deserialize(std::string_view json_data, StructureObject &value, memory::resource_pointer mr, glz::error_ctx &out_ec)`
- **参数**: `json_data` — JSON 原始数据；`value` — 接收数据的对象引用；`mr` — 缓冲区使用的内存资源；`out_ec` — 用于存储反序列化错误上下文
- **返回值**: `bool` — true 反序列化成功，false 反序列化失败
- **调用（向下）**: `glz::read_json()`、`memory::current_resource()`
- **被调用（向上）**: 需要 PMR 内存资源的反序列化场景
- **知识域**: JSON 反序列化（PMR 分配）

### 函数模板: deserialize() [带内存资源，忽略错误]

- **功能说明**: 将 JSON 字符串反序列化为对象，使用指定的内存资源，忽略错误上下文。
- **签名**: `template <typename StructureObject> [[nodiscard]] bool deserialize(std::string_view json_data, StructureObject &value, memory::resource_pointer mr)`
- **参数**: `json_data` — JSON 原始数据；`value` — 接收数据的对象引用；`mr` — 缓冲区使用的内存资源
- **返回值**: `bool` — true 反序列化成功，false 反序列化失败
- **调用（向下）**: `deserialize(json_data, value, mr, out_ec)`
- **被调用（向上）**: 不关心错误细节的反序列化场景
- **知识域**: JSON 反序列化（简化版）

---

## transformer 概述

Transformer 模块的详细设计文档。基于 glaze 库的编译期反射 JSON 序列化系统。`serialize` 提供 3 个重载（基础、带错误上下文、带内存资源），`deserialize` 提供 4 个重载（基础、带错误上下文、带内存资源+错误、带内存资源忽略错误）。所有函数均为模板，支持任意 glaze 注册类型。

## 相关页面

- [[memory/container|container]] — PMR 容器，serialize/deserialize 使用 `memory::string` 和 `memory::resource_pointer`
- [[loader/load|load]] — 配置加载器，调用 `transformer::json::deserialize()` 解析 JSON 配置
- [[agent/config|config]] — Agent 配置结构体，通过 glaze 编译期反射实现自动序列化
- [[stealth/registry|registry]] — TLS 伪装方案注册，配置结构体通过 glaze 反序列化
