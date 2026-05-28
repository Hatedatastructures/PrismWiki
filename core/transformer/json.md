---
tags: [transformer, json]
layer: core
source: include/prism/transformer/json.hpp
title: Transformer Json
module: transformer
updated: 2026-05-27
---

# Transformer Json

基于 glaze 库的 JSON 序列化与反序列化。

## 核心接口

### 序列化

| 重载 | 返回值 | 说明 |
|------|--------|------|
| `serialize(obj, out)` | `bool` | 结果写入 `memory::string&` |
| `serialize(obj, out, ec)` | `bool` | 附带错误上下文 |
| `serialize(obj, mr)` | `memory::string` | 返回 PMR 字符串 |

### 反序列化

| 重载 | 返回值 | 说明 |
|------|--------|------|
| `deserialize(json, obj)` | `bool` | 直接反序列化 |
| `deserialize(json, obj, ec)` | `bool` | 附带错误上下文 |
| `deserialize(json, obj, mr)` | `bool` | 指定 PMR 容器的内存资源 |

## 设计决策

### 为什么 deserialize 用临时对象 + move？

glaze 的 `read_json` 在解析失败时会破坏目标对象的状态。对于支持移动构造的类型，先解析到临时对象，成功后再 `move` 到目标，保证失败时目标对象不变。

**后果**: 不支持移动构造的类型（如某些 C 数组）直接解析，失败时状态可能被破坏。

### 为什么用 glaze 而非 nlohmann/json？

glaze 是编译期反射 + 零拷贝设计，反序列化直接写入结构体字段，不需要中间 JSON 树。编译期反射在 debug 构建下有更好的错误提示。性能优于运行时反射方案。

**后果**: 类型必须定义 `glz::meta` 特化或使用 `glz::object` 宏注册字段映射。

## 约束

### PMR 容器类型必须指定内存资源

**类型**: 调用顺序

**规则**: 包含 `memory::vector` 等 PMR 容器的类型，反序列化时必须通过 `deserialize(json, obj, mr)` 指定内存资源。否则 PMR 容器使用默认分配器（null_resource），写入时崩溃。

**违反后果**: 运行时崩溃（null_resource dereference）。

**源码依据**: `transformer/json.hpp`

## 类型映射定义

类型需定义 `glz::meta` 特化：

```cpp
struct my_type {
    int port;
    memory::string host;
    struct glaze {
        using T = my_type;
        static constexpr auto value = glz::object("port", &T::port, "host", &T::host);
    };
};
```

## 引用关系

### 被引用

- [[core/loader/load|loader::load]]：JSON 配置反序列化
- [[core/instance/serialize|instance::serialize]]：运行时状态序列化
