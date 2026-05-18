---
layer: core
source: prism/transformer/json.hpp
title: Transformer Json
---

# Transformer Json

JSON 序列化与反序列化功能，基于 glaze 库实现。

## 源码位置

`I:/code/Prism/include/prism/transformer/json.hpp`

## 序列化接口

### serialize - 基础版本

```cpp
template <typename StructureObject>
[[nodiscard]] bool serialize(const StructureObject &value, memory::string &out);
```

将对象序列化为JSON字符串，结果存入 `out`。

```cpp
config cfg;
memory::string json;
if (json::serialize(cfg, json)) {
    std::cout << json;
}
```

### serialize - 带错误上下文

```cpp
template <typename StructureObject>
[[nodiscard]] bool serialize(const StructureObject &value, 
                             memory::string &out, 
                             glz::error_ctx &out_ec);
```

返回详细错误上下文。

### serialize - 带内存资源

```cpp
template <typename StructureObject>
[[nodiscard]] memory::string serialize(const StructureObject &value,
                                       memory::resource_pointer mr = memory::current_resource());
```

返回PMR字符串，可直接指定内存资源。

```cpp
auto json = json::serialize(cfg, memory::system::thread_local_pool());
```

## 反序列化接口

### deserialize - 基础版本

```cpp
template <typename StructureObject>
[[nodiscard]] bool deserialize(std::string_view json_data, StructureObject &value);
```

对于支持移动构造的类型，使用临时对象避免破坏原对象状态。

```cpp
config cfg;
std::string_view json = "{...}";
if (json::deserialize(json, cfg)) {
    // 使用cfg
}
```

### deserialize - 带错误上下文

```cpp
template <typename StructureObject>
[[nodiscard]] bool deserialize(std::string_view json_data,
                               StructureObject &value,
                               glz::error_ctx &out_ec);
```

### deserialize - 带内存资源

```cpp
template <typename StructureObject>
[[nodiscard]] bool deserialize(std::string_view json_data,
                               StructureObject &value,
                               memory::resource_pointer mr);
```

支持PMR容器的类型（如 `std::pmr::vector`）可指定内存资源：

```cpp
memory::vector<int> nums(memory::system::thread_local_pool());
json::deserialize("[1,2,3]", nums, memory::system::thread_local_pool());
```

## 元数据定义

序列化类型需定义 `glz::meta` 特化或使用 `GLZ_META` 宏：

```cpp
struct config {
    int port;
    std::string host;
    
    // 方式1: glz::meta特化
    struct glaze {
        using T = config;
        static constexpr auto value = glz::object(
            "port", &T::port,
            "host", &T::host
        );
    };
};

// 方式2: GLZ_META宏
struct config {
    int port;
    std::string host;
    GLZ_META(config, port, host);
};
```

## 错误处理

```cpp
memory::string json;
glz::error_ctx ec;
if (!json::serialize(value, json, ec)) {
    trace::error("序列化失败: {}", ec.error_code);
}

if (!json::deserialize(json_data, value, ec)) {
    trace::error("反序列化失败: {}", ec.error_code);
}
```

## 调用链

```mermaid
graph TD
    A[json::serialize] --> B[out.clear]
    A --> C[glz::write_json]
    C --> D{有错误?}
    D -->|Yes| E[out.clear, return false]
    D -->|No| F[return true]
    
    G[json::deserialize] --> H{支持移动构造?}
    H -->|Yes| I[创建临时对象]
    I --> J[glz::read_json]
    J --> K[value = move(temp)]
    H -->|No| L[直接glz::read_json]
```

## 使用示例

```cpp
// 序列化配置
psm::config cfg;
cfg.port = 8080;
cfg.host = "localhost";

memory::string json_str;
if (json::serialize(cfg, json_str)) {
    trace::info("配置JSON: {}", json_str);
}

// 反序列化配置
std::string_view json_data = "{\"port\":443,\"host\":\"example.com\"}";
psm::config parsed;
if (json::deserialize(json_data, parsed)) {
    trace::info("解析端口: {}", parsed.port);
}

// 使用内存池
auto json = json::serialize(cfg, memory::system::global_pool());
```

## 相关页面

- [[core/transformer/overview]] - Transformer模块总览
- [[core/memory/container]] - PMR字符串
- [[core/loader/load]] - 配置加载器使用JSON