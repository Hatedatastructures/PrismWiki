---
title: "registry — 伪装方案注册表"
source: "include/prism/stealth/registry.hpp"
module: "stealth"
type: api
tags: [stealth, registry, 注册表, 单例]
created: 2026-05-15
updated: 2026-05-15
related:
  - "[[stealth/executor|executor]]"
  - "[[stealth/scheme|scheme]]"
  - "[[stealth/reality/scheme|reality/scheme]]"
  - "[[stealth/native|native]]"
---

# registry.hpp

> 源码: `include/prism/stealth/registry.hpp`
> 实现: `src/prism/stealth/registry.cpp`
> 模块: [[stealth|stealth]]

## 概述

伪装方案注册表。单例模式，管理所有 `stealth_scheme` 的注册和查询。启动阶段通过 `register_all_schemes()` 手动注册所有方案，运行时只读，无需同步。

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[stealth/scheme|scheme]] | 使用 stealth_scheme 基类和 shared_scheme 类型别名 |
| 被依赖 | [[stealth/executor|executor]] | 执行器从注册表获取方案列表 |
| 被依赖 | [[loader/load|main]] | 启动时调用 register_all_schemes() |
| 被依赖 | [[recognition/recognition|recognition]] | 识别模块查询方案列表 |

## 命名空间

`psm::stealth`

---

## 类: scheme_registry

> 源码: `include/prism/stealth/registry.hpp:30`

### 概述

伪装方案注册表（单例）。管理所有 `stealth_scheme` 的注册和查询。启动时通过 `register_all_schemes()` 注册所有方案，运行时由 [[recognition/recognition|recognition]] 模块查询和使用。

### 设计意图

注册表采用单例模式，确保全局唯一。启动阶段注册所有方案，运行时只读，无需同步。

**使用流程**：
1. 启动阶段调用 `register_all_schemes()`
2. [[recognition/recognition|recognition]] 调用 `scheme_registry::instance().all()` 获取所有方案
3. 对每个方案调用 `detect()` 获取候选列表
4. 通过 [[stealth/executor|scheme_executor]] 执行候选方案

### 成员变量

| 变量 | 类型 | 说明 |
|------|------|------|
| schemes_ | std::vector\<shared_scheme\> | 方案列表，按注册顺序排列 |

### 成员函数一览

| 函数 | 功能简述 | 详见 |
|------|----------|------|
| instance() | 获取全局单例 | 下文 |
| add() | 注册方案 | 下文 |
| all() | 获取所有已注册的方案 | 下文 |
| find() | 按名称查找方案 | 下文 |

### 生命周期

1. **构造**: 程序启动时自动构造（静态局部变量）
2. **使用**: 被 [[recognition/recognition|recognition]] 模块查询和使用
3. **销毁**: 程序退出时自动销毁

### 线程安全

- 启动阶段注册方案，运行时只读，线程安全

### 异常安全

- 所有函数不抛异常

---

## 函数: instance()

> 源码: `include/prism/stealth/registry.hpp:37`
> 实现: `src/prism/stealth/registry.cpp:34`

### 功能

获取全局单例。返回 `scheme_registry` 的全局唯一实例，使用 Meyers' Singleton（静态局部变量）实现线程安全的延迟初始化。

### 签名

```cpp
static auto instance() -> scheme_registry &;
```

### 参数

无

### 返回值

`scheme_registry &` — 全局单例引用

### 调用（向下）

- 无

### 被调用（向上）

- register_all_schemes() — 注册方案时获取单例
- [[stealth/executor|scheme_executor]] — 构建执行器时获取注册表
- [[recognition/recognition|recognize]] — 识别模块获取方案列表

### 知识域

- 设计模式 — Meyers' Singleton 延迟初始化

---

## 函数: add()

> 源码: `include/prism/stealth/registry.hpp:44`
> 实现: `src/prism/stealth/registry.cpp:40`

### 功能

注册方案。将方案实例添加到注册表的 `schemes_` 列表末尾。启动阶段调用，运行时不再修改。注册顺序决定默认优先级。

### 签名

```cpp
auto add(shared_scheme scheme) -> void;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| scheme | shared_scheme | 方案实例（std::shared_ptr\<stealth_scheme\>） |

### 返回值

无

### 调用（向下）

- 无（std::vector::push_back）

### 被调用（向上）

- register_all_schemes() — 注册所有方案

### 知识域

- [[stealth/scheme|shared_scheme]] — 方案智能指针类型别名

---

## 函数: all()

> 源码: `include/prism/stealth/registry.hpp:50`
> 实现: `src/prism/stealth/registry.cpp:45`

### 功能

获取所有已注册的方案。返回方案列表的常量引用，按注册顺序排列（即默认优先级）。

### 签名

```cpp
[[nodiscard]] auto all() const -> const std::vector<shared_scheme> &;
```

### 参数

无

### 返回值

`const std::vector<shared_scheme> &` — 方案列表常量引用

### 调用（向下）

- 无

### 被调用（向上）

- [[stealth/executor|scheme_executor]] — 执行器构造时拷贝方案列表
- [[recognition/recognition|recognize]] — 识别模块获取方案列表进行检测

### 知识域

- [[stealth/scheme|stealth_scheme]] — 方案基类

---

## 函数: find()

> 源码: `include/prism/stealth/registry.hpp:57`
> 实现: `src/prism/stealth/registry.cpp:50`

### 功能

按名称查找方案。使用 `std::ranges::find_if` 在 `schemes_` 列表中查找指定名称的方案，未找到返回 nullptr。

### 签名

```cpp
[[nodiscard]] auto find(std::string_view name) const -> shared_scheme;
```

### 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| name | std::string_view | 方案名称（如 "reality"、"native"） |

### 返回值

`shared_scheme` — 找到的方案智能指针，未找到返回 nullptr

### 调用（向下）

- [[stealth/scheme|stealth_scheme::name()]] — 获取方案名称用于比较

### 被调用（向上）

- [[stealth/executor|find_scheme]] — 执行器查找方案（执行器内部有独立实现）

### 知识域

- [[stealth/scheme|stealth_scheme]] — 方案基类

---

## 函数: register_all_schemes()

> 源码: `include/prism/stealth/registry.hpp:68`
> 实现: `src/prism/stealth/registry.cpp:17`

### 功能

注册所有伪装方案。在 `main()` 或启动阶段调用，按优先级顺序注册 reality/shadowtls/restls/anytls/trusttunnel/native 等方案。注册顺序即为默认优先级：Layer 0 reality（独占特征）→ Layer 1 shadowtls（HMAC 验证）→ Layer 2 restls/anytls/trusttunnel（模糊匹配）→ 兜底 native。新增方案只需在此函数中添加一行。

### 签名

```cpp
auto register_all_schemes() -> void;
```

### 参数

无

### 返回值

无

### 调用（向下）

- scheme_registry::instance() — 获取全局单例
- scheme_registry::add() — 注册每个方案
- [[stealth/reality/scheme|reality/scheme]] — 注册 Reality 方案
- [[stealth/shadowtls/scheme|shadowtls/scheme]] — 注册 ShadowTLS 方案
- [[stealth/restls/scheme|restls/scheme]] — 注册 Restls 方案
- [[stealth/anytls/scheme|anytls/scheme]] — 注册 AnyTLS 方案
- [[stealth/trusttunnel/scheme|trusttunnel/scheme]] — 注册 TrustTunnel 方案
- [[stealth/native|native]] — 注册 Native 兜底方案

### 被调用（向上）

- [[loader/load|main]] — 启动时调用

### 知识域

- [[stealth/scheme|stealth_scheme]] — 方案基类和分层检测架构
