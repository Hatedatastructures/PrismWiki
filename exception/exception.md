---
title: Exception 模块
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [exception, error-handling, infrastructure]
related: [[fault/overview], [fault/detail], [loader/overview], [agent/overview]]
---

# Exception 模块

## 1. 模块职责与边界

Exception 模块定义 Prism 的**异常层次结构**，仅用于启动阶段或致命错误。热路径必须使用错误码（[[fault]]）。

**异常层次**：
- **基类 `deviant`**：继承 `std::runtime_error`，存储 `std::error_code` + `source_location`
- **子类**：`network`（网络异常）、`protocol`（协议异常）、`security`（安全异常）

**边界**：Exception 模块是 header-only，无源文件。

## 2. 关键类和接口

### 2.1 `deviant` — 异常基类（`deviant.hpp`）

```cpp
namespace psm::exception {
    class deviant : public std::runtime_error {
    public:
        // 主构造函数（错误码 + 可选描述）
        explicit deviant(std::error_code ec, std::string_view desc = {},
                         const std::source_location &loc = std::source_location::current());

        // 向后兼容字符串构造
        explicit deviant(const std::string &msg,
                         const std::source_location &loc = std::source_location::current());

        // 带格式化参数
        template<typename... Args>
        explicit deviant(const std::source_location &loc, std::format_string<Args...> fmt, Args&&... args);

        [[nodiscard]] const std::error_code &error_code() const noexcept;
        [[nodiscard]] const std::source_location &location() const noexcept;
        [[nodiscard]] std::string filename() const;
        [[nodiscard]] virtual std::string dump() const;
        // dump() 输出格式: [filename:line] [TYPE:value] description

    protected:
        [[nodiscard]] virtual std::string_view type_name() const noexcept = 0;

    private:
        std::error_code ec_;
        std::source_location location_;
    };
}
```

**关键设计**：
- `source_location::current()` 自动捕获抛出点的文件名、行号、列号
- 错误码构造是首选方式，字符串构造自动转为 `generic_error`
- `dump()` 生成 `[filename:line] [TYPE:value] description` 格式

### 2.2 `network` — 网络异常（`network.hpp`）

```cpp
class network : public deviant {
public:
    explicit network(fault::code err, const std::source_location &loc = ...);
    explicit network(fault::code err, std::string_view desc, const std::source_location &loc = ...);
    explicit network(const std::string &msg, const std::source_location &loc = ...);
    template<typename... Args> explicit network(std::format_string<Args...> fmt, Args&&... args);
protected:
    std::string_view type_name() const noexcept override { return "NETWORK"; }
};
```

### 2.3 `protocol` — 协议异常（`protocol.hpp`）

```cpp
class protocol : public deviant {
public:
    explicit protocol(fault::code err, const std::source_location &loc = ...);
    explicit protocol(fault::code err, std::string_view desc, const std::source_location &loc = ...);
    explicit protocol(const std::string &msg, const std::source_location &loc = ...);
    template<typename... Args> explicit protocol(std::format_string<Args...> fmt, Args&&... args);
protected:
    std::string_view type_name() const noexcept override { return "PROTOCOL"; }
};
```

### 2.4 `security` — 安全异常（`security.hpp`）

```cpp
class security : public deviant {
public:
    explicit security(fault::code err, const std::source_location &loc = ...);
    explicit security(fault::code err, std::string_view desc, const std::source_location &loc = ...);
    explicit security(const std::string &msg, const std::source_location &loc = ...);
    template<typename... Args> explicit security(std::format_string<Args...> fmt, Args&&... args);
protected:
    std::string_view type_name() const noexcept override { return "SECURITY"; }
};
```

## 3. 文件清单

### 头文件（`include/prism/exception/`）

```
exception.hpp            # 聚合头文件
├── deviant.hpp          # 异常基类（deviant）
├── network.hpp          # 网络异常（network）
├── protocol.hpp         # 协议异常（protocol）
└── security.hpp         # 安全异常（security）
```

**注意**：Exception 模块是 header-only，无源文件。

## 4. 异常层次结构

```
std::runtime_error
└── exception::deviant (抽象基类)
    ├── exception::network    (type_name = "NETWORK")
    │   └── 用途: 网络配置和初始化阶段错误
    ├── exception::protocol   (type_name = "PROTOCOL")
    │   └── 用途: 协议检测、握手、解析阶段错误
    └── exception::security   (type_name = "SECURITY")
        └── 用途: 安全配置和初始化阶段错误
```

## 5. 设计原理

### 5.1 双轨错误处理
- **热路径**：使用 `fault::code` 返回错误码，零分配、零异常开销
- **启动/致命错误**：使用异常层次结构，提供丰富的诊断信息

### 5.2 源码位置捕获
`std::source_location::current()` 在构造时自动捕获，无需手动传递 `__FILE__` / `__LINE__`。

### 5.3 结构化错误信息
`deviant` 核心存储 `std::error_code`（与 [[fault]] 集成），而非纯字符串。这使得异常可被程序化处理（检查错误码值），同时保留人类可读的描述。

### 5.4 向后兼容
字符串和格式化构造函数自动转为 `generic_error` 错误码，建议迁移到错误码构造函数以保留完整的错误分类信息。

## 相关页面

- [[fault]] — Fault 模块概述
- [[fault]] — Fault 模块详细设计
- [[loader]] — Loader 模块（抛出异常）
- [[architecture]] — 架构设计
