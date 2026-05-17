---
title: PMR 内存策略
description: Prism 多态内存资源 (PMR) 策略，实现热路径零堆分配
---

# PMR 内存策略

Prism 使用 **PMR (Polymorphic Memory Resource)** 实现热路径零堆分配，确保高性能内存管理。

## 核心概念

### 什么是 PMR

PMR (Polymorphic Memory Resource) 是 C++17 引入的多态内存资源抽象，允许容器在运行时选择内存分配策略，而无需修改容器类型。

### 为什么使用 PMR

- **零堆分配**: 热路径使用预分配内存池，避免动态内存分配
- **缓存友好**: 内存竞技场提供连续内存布局
- **确定性释放**: 批量释放内存，减少碎片

## 内存类型

### 全局池字符串

`memory::string` 使用全局内存池：

```cpp
namespace psm::memory {
    using string = std::pmr::string;
}
```

特点：
- 进程生命周期
- 线程安全
- 适用于配置、域名等长生命周期字符串

### 帧竞技场向量

`memory::vector<T>` 使用帧竞技场：

```cpp
namespace psm::memory {
    template<typename T>
    using vector = std::pmr::vector<T>;
}
```

特点：
- 请求生命周期
- 单线程使用
- 适用于临时缓冲区、数据包处理

## 内存池实现

### 全局内存池

启动时必须调用 `memory::system::enable_global_pooling()` 初始化：

```cpp
// main.cpp
int main() {
    // 初始化全局内存池
    psm::memory::system::enable_global_pooling();

    // ... 其他初始化
}
```

### 帧竞技场

每个请求或会话使用帧竞技场：

```cpp
auto handle_request(tcp::socket socket) -> net::awaitable<void> {
    // 创建帧竞技场
    psm::memory::arena arena;

    // 使用竞技场分配向量
    psm::memory::vector<std::byte> buffer(&arena);

    // 请求结束，竞技场自动释放所有内存
}
```

## Memory 模块结构

```
include/prism/memory/
├── container.hpp   // PMR 容器定义 (string, vector 等)
└── pool.hpp        // 内存池实现
```

### container.hpp

定义 PMR 容器别名：

```cpp
namespace psm::memory {
    // 使用全局池的字符串
    using string = std::pmr::string;

    // 使用帧竞技场的向量
    template<typename T>
    using vector = std::pmr::vector<T>;

    // 其他容器...
}
```

### pool.hpp

内存池实现：

```cpp
namespace psm::memory::system {
    // 启用全局内存池
    void enable_global_pooling();
}

namespace psm::memory {
    // 帧竞技场
    class arena {
    public:
        arena();
        ~arena();

        // 分配内存
        void* allocate(size_t size, size_t alignment = alignof(std::max_align_t));

        // 重置竞技场（释放所有分配）
        void reset();
    };
}
```

## 使用指南

### 字符串使用

```cpp
// 配置和长生命周期字符串
memory::string domain = "example.com";
memory::string config_value = some_config;

// 避免频繁转换
std::string std_str = "temp";
memory::string pmr_str(std_str);  // 有拷贝开销
```

### 向量使用

```cpp
// 在协程中使用帧竞技场
auto process_data(buffer_span input) -> net::awaitable<void> {
    memory::arena arena;
    memory::vector<std::byte> buffer(&arena);

    buffer.reserve(input.size());
    std::copy(input.begin(), input.end(), std::back_inserter(buffer));

    // 处理数据...
}
```

### 内存池选择

| 场景 | 推荐类型 | 原因 |
|------|----------|------|
| 配置字符串 | `memory::string` | 长生命周期，全局共享 |
| 域名缓存 | `memory::string` | 长生命周期 |
| 临时缓冲区 | `memory::vector<T>` | 短生命周期，帧分配 |
| 数据包处理 | `memory::vector<std::byte>` | 批量释放高效 |

## 初始化流程

```cpp
// main.cpp 启动顺序
int main() {
    // 1. 初始化全局内存池
    psm::memory::system::enable_global_pooling();

    // 2. 注册伪装方案
    psm::stealth::register_all_schemes();

    // 3. 加载配置
    auto config = psm::loader::load("config.json");

    // 4. 初始化日志
    psm::trace::init(config.trace);

    // ...
}
```

## 性能考量

- **避免混合使用**: 不要在热路径中混合 PMR 和标准分配器
- **预分配**: 对于已知大小的容器，使用 `reserve()` 预分配
- **竞技场大小**: 根据实际负载调整帧竞技场初始大小

## 相关文档

- [[overview]] - 编码规范概览
- [[/dev/pmr-memory-pool]] - PMR 内存池详细文档
- [[/dev/performance/benchmark]] - 性能基准测试