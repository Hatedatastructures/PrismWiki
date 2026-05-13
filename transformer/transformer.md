---
title: Transformer 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [transformer, json, serialization, infrastructure]
related: [[memory], [loader], [architecture]]
module: transformer
category: infrastructure
layer: serialization
status: stable
dependencies:
  - glaze
  - memory
dependents:
  - loader
  - agent
---

# Transformer 模块

## 1. 模块定位

Transformer 模块是 Prism 的**数据序列化层**，提供基于 glaze 库的 JSON 序列化和反序列化功能。它是外部配置文件与内部 C++ 结构体之间的桥梁。

**核心价值**：类型安全、PMR 集成的 JSON 序列化接口，隔离 glaze 库的实现细节。

**边界**：Transformer 模块是 header-only 的纯序列化层，不包含业务逻辑。使用前需为序列化类型定义 `glz::meta` 特化或使用 `GLZ_META` 宏。

## 2. 核心功能

### 2.1 JSON 序列化

`transformer::json::serialize<T>()` 将对象序列化为 JSON 字符串：

```cpp
// 基本用法
bool ok = transformer::json::serialize(obj, out_string);

// 带错误上下文
glz::error_ctx ec;
bool ok = transformer::json::serialize(obj, out_string, ec);

// 返回 PMR 字符串
memory::string json = transformer::json::serialize(obj);
```

### 2.2 JSON 反序列化

`transformer::json::deserialize<T>()` 将 JSON 字符串解析为对象：

```cpp
// 基本用法
bool ok = transformer::json::deserialize(json_data, obj);

// 带错误上下文
glz::error_ctx ec;
bool ok = transformer::json::deserialize(json_data, obj, ec);

// 带内存资源
bool ok = transformer::json::deserialize(json_data, obj, memory_resource);
```

### 2.3 安全反序列化

反序列化使用临时对象策略：先解析到临时对象，成功后通过移动赋值覆盖原对象。这避免了部分解析导致的对象状态破坏。

## 3. 关键类和接口

### 3.1 `serialize<T>()` — JSON 序列化（`json.hpp`）

```cpp
namespace psm::transformer::json {
    // 基本序列化（输出到已有字符串）
    template<typename T>
    [[nodiscard]] bool serialize(const T &value, memory::string &out);

    // 带错误上下文
    template<typename T>
    [[nodiscard]] bool serialize(const T &value, memory::string &out, glz::error_ctx &out_ec);

    // 返回 PMR 字符串（可选内存资源）
    template<typename T>
    [[nodiscard]] memory::string serialize(const T &value,
        memory::resource_pointer mr = memory::current_resource());
}
```

**实现要点**：输出前调用 `out.clear()` 清空，使用 `if constexpr` 区分 glaze 返回类型，失败时清空输出字符串。

### 3.2 `deserialize<T>()` — JSON 反序列化（`json.hpp`）

```cpp
namespace psm::transformer::json {
    template<typename T>
    [[nodiscard]] bool deserialize(std::string_view json_data, T &value);

    template<typename T>
    [[nodiscard]] bool deserialize(std::string_view json_data, T &value, glz::error_ctx &out_ec);

    template<typename T>
    [[nodiscard]] bool deserialize(std::string_view json_data, T &value,
        memory::resource_pointer mr, glz::error_ctx &out_ec);

    template<typename T>
    [[nodiscard]] bool deserialize(std::string_view json_data, T &value, memory::resource_pointer mr);
}
```

**安全反序列化策略**：
1. 对于支持默认构造 + 移动赋值的类型：先解析到临时对象，成功后 `value = std::move(temp)`
2. 对于不支持的类型：直接解析到目标对象
3. 带内存资源的版本：使用 `T(mr)` 构造临时对象（如果类型支持）

### 3.3 PMR 内存资源集成

带 `memory::resource_pointer` 参数的 `deserialize` 重载：
- 如果类型支持 `T(memory::resource_pointer)` 构造，使用该构造器创建临时对象
- 否则回退到不带内存资源的版本
- `mr` 为 `nullptr` 时使用 `memory::current_resource()`

## 4. 使用模式

### 4.1 配置文件加载（典型场景）
```cpp
config cfg;
memory::string content = read_file(path);
bool ok = transformer::json::deserialize(
    std::string_view{content.data(), content.size()}, cfg);
```

### 4.2 带错误诊断的序列化
```cpp
glz::error_ctx ec;
memory::string output;
if (!transformer::json::serialize(obj, output, ec)) {
    trace::error("JSON serialize failed: {}", ec.ec);
}
```

## 5. 文件清单

### 头文件（`include/prism/transformer/`）

```
transformer.hpp          # 聚合头文件
└── json.hpp             # JSON 序列化/反序列化（serialize, deserialize）
```

**注意**：Transformer 模块是 header-only，无源文件。

## 6. 设计原理

- **glaze 库封装**：对外只暴露简洁的 `serialize` / `deserialize` 接口，glaze 头文件不泄漏到业务层
- **编译时类型分发**：使用 `if constexpr` 区分 glaze 的不同返回类型，确保版本兼容性
- **安全反序列化**：临时对象策略避免 JSON 解析失败时破坏原对象状态
- **PMR 集成**：`serialize()` 返回 `memory::string`，`deserialize()` 支持传入 `memory::resource_pointer`

## 7. 外部依赖

| 依赖 | 用途 | 集成方式 |
|------|------|---------|
| glaze | 高性能 JSON 序列化 | FetchContent |

## 相关页面

- [[loader]] — Loader 模块（主要使用者）
- [[memory]] — Memory 模块
- [[architecture]] — 架构设计
