---
title: 编码规范概览
description: Prism 项目的整体编码规范与约定，涵盖命名、协程、内存管理等方面
layer: dev
---

# 编码规范概览

Prism 采用 C++23 标准，遵循严格的编码规范以确保代码质量、性能和可维护性。本文档概述编码规范的核心要点。

## 核心原则

Prism 项目的编码规范围绕以下核心原则展开：

1. **一致性** - 统一的命名风格和代码结构
2. **性能优先** - 热路径零堆分配，协程纯度保证
3. **安全可靠** - 生命周期安全管理，双轨错误处理
4. **可维护性** - 清晰的模块边界和文档注释

## 规范体系

| 规范 | 说明 | 文档 |
|------|------|------|
| 命名规范 | 命名空间、文件、类型命名约定 | [[naming]] |
| 协程约定 | awaitable、co_await、co_spawn 使用 | [[coroutine]] |
| PMR 内存策略 | 内存池、帧竞技场、容器选择 | [[pmr]] |
| 生命周期安全 | co_spawn 捕获、迭代器失效 | [[lifecycle]] |
| 错误处理 | 错误码与异常双轨策略 | [[error]] |
| Doxygen 注释 | 文档注释风格与标准 | [[doxygen]] |

## 命名空间约定

所有代码使用 `psm::` 作为根命名空间：

```cpp
namespace psm::resolve { /* DNS 解析 */ }
namespace psm::channel { /* 连接通道 */ }
namespace psm::recognition { /* 协议识别 */ }
namespace psm::agent { /* 代理核心 */ }
```

## 关键别名

```cpp
namespace net = boost::asio;  // Boost.Asio 别名
```

## 相关文档

- [[naming]] - 命名规范详解
- [[coroutine]] - 协程约定与禁止事项
- [[pmr]] - PMR 内存管理策略
- [[lifecycle]] - 生命周期安全指南
- [[error]] - 错误处理策略
- [[doxygen]] - Doxygen 注释规范
- [[dev/modules|modules]] - 模块结构概览

## 为什么需要编码规范

Prism 是一个高性能 C++23 代理软件项目，编码规范的存在不仅仅是为了"代码好看"，而是解决以下核心问题：

### 1. 降低认知负担

当所有代码遵循同一套规则时，阅读者不需要"翻译"每个开发者的个人风格。看到一个 `snake_case` 函数名就能确定它是生产代码，看到 `PascalCase` 就知道是测试函数。

### 2. 防止性能退化

没有规范的代码容易引入隐性性能问题：热路径中的隐式堆分配、协程中 `std::optional` 的不当使用、缺乏 `[[nodiscard]]` 导致的错误值被忽略等。编码规范将这些常见陷阱前置到开发阶段就消除。

### 3. 安全与生命周期管理

C++ 中最危险的 Bug 来源是生命周期管理错误。`co_spawn` 的悬垂捕获、迭代器失效、引用越界等问题，通过规范中的强制规则（如"禁止裸 `co_spawn`"）可以在编译期或 Review 期就发现。

### 4. 加速 Code Review

规范提供了客观的审查标准。Review 不再讨论"风格偏好"，而是检查清单式的"是否违反 X 规则"。这显著减少了 Review 时间。

## 编码原则详解

### 命名原则

- **见名知意**: 名称应当自解释，避免缩写和歧义
- **一致性**: 同类型元素使用统一的命名模式
- **区分度**: 避免容易混淆的命名（如 `conn` vs `con`）

### 注释原则

- **解释 Why 而非 What**: 代码已经表达了"做什么"，注释应解释"为什么这样做"
- **Doxygen 标准**: 公共 API 必须有 Doxygen 注释
- **更新同步**: 代码变更必须同步更新注释

### 异常原则

- **双轨策略**: 可恢复错误使用 `expected<T, ErrorCode>`，不可恢复错误使用 `throw`
- **热路径无异常**: 热路径代码不使用异常机制（零成本原则）

### 协程原则

- **禁止裸 `co_spawn`**: 所有 `co_spawn` 必须有生命周期管理
- **awaitable 返回**: 异步函数一律返回 `net::awaitable<T>`
- **禁止 co_await 后使用悬垂引用**: 注意协程挂起后的生命周期

### 内存管理原则

- **热路径零堆分配**: 热路径使用 PMR 内存池
- **帧竞技场**: 协程帧使用 arena 分配器
- **容器选择**: 优先 `std::vector` + 预留，避免不必要的 `std::shared_ptr`

## 示例与反例对比

### 命名示例

**正确：**
```cpp
auto create_connection_pool(size_t capacity) -> net::awaitable<connection_pool>;
auto is_domain_blocked(std::string_view domain) -> bool;
```

**错误：**
```cpp
auto cp(size_t c) -> net::awaitable<CP>;              // 缩写不明
auto chk(std::string_view d) -> bool;                 // 缩写不明
auto GetConnectionPool() -> net::awaitable<ConnectionPool>;  // PascalCase 不应用于生产代码
```

### 协程示例

**正确：**
```cpp
// 生命周期安全的 co_spawn
auto start_dns_server(net::io_context& ctx) -> net::awaitable<void> {
    auto handle = co_await net::co_spawn(
        ctx,
        dns_server::run(std::move(config)),
        net::use_awaitable
    );
    // handle 确保清理
}
```

**错误：**
```cpp
// 裸 co_spawn，无生命周期管理
void start_dns_server(net::io_context& ctx) {
    net::co_spawn(ctx, dns_server::run(config), net::detached);  // 危险！config 可能被销毁
}
```

### 错误处理示例

**正确：**
```cpp
[[nodiscard]] auto resolve_domain(std::string_view domain)
    -> net::awaitable<expected<ip_address, fault::resolve_error>>;
```

**错误：**
```cpp
// 未标记 nodiscard，调用方可能忽略错误
auto resolve_domain(std::string_view domain)
    -> net::awaitable<ip_address>;  // 解析失败时怎么办？
```

## 代码审查检查清单

### 命名审查

- [ ] 所有类型是否使用 `snake_case`？
- [ ] 测试函数是否使用 `PascalCase`？
- [ ] 文件命名是否为 `snake_case.hpp`？
- [ ] 命名是否自解释，无歧义缩写？

### 协程审查

- [ ] `co_spawn` 是否有生命周期管理？
- [ ] `co_await` 后是否有悬垂引用风险？
- [ ] 异步函数是否返回 `net::awaitable<T>`？
- [ ] 是否使用了禁止的协程模式？

### 内存审查

- [ ] 热路径是否有隐式堆分配？
- [ ] 容器是否预留了容量（`reserve`）？
- [ ] 是否不必要地使用了 `shared_ptr`？

### 错误处理审查

- [ ] 返回值是否标记 `[[nodiscard]]`？
- [ ] 错误码类型是否统一？
- [ ] 热路径是否避免了异常？

### 注释审查

- [ ] 公共 API 是否有 Doxygen 注释？
- [ ] 注释是否解释了 "Why"？
- [ ] 代码变更后注释是否同步更新？

### 性能审查

- [ ] 热路径是否零堆分配？
- [ ] 是否使用了不必要的虚函数？
- [ ] 循环内是否有不必要的对象构造？