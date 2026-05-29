---
tags: [exception, deviant]
layer: core
module: exception
source: include/prism/exception/deviant.hpp
title: exception::deviant
---

# exception::deviant

项目异常基类（抽象），所有自定义异常的根。核心存储 `std::error_code`，自动捕获源码位置。

## 接口

| 方法 | 说明 |
|------|------|
| `error_code()` | 返回 `const std::error_code&` |
| `location()` | 返回 `const std::source_location&` |
| `filename()` | 返回文件名（不含路径） |
| `dump()` | 格式化输出 `[file:line] [TYPE:value] desc` |
| `type_name()` | 纯虚函数，子类实现（如 `"SECURITY"`） |

## 设计决策

### 为什么 deviant 用 error_code 而非 fault::code 枚举成员？

`std::error_code` 是标准库通用错误表示，`deviant` 继承 `std::runtime_error`，`what()` 消息由 `ec.message()` 生成。使用 `error_code` 可与标准库 `catch` 和日志系统集成，同时通过 `fault::category()` 保留自定义错误分类。

**后果**: 构造时需调用 `fault::make_error_code()` 将枚举转为 `error_code`。

### 为什么有向后兼容的字符串构造函数？

历史遗留。早期版本直接用字符串构造异常。字符串构造自动映射为 `generic_error` 错误码，不推荐使用——新代码应始终传 `fault::code`。

**后果**: 字符串构造的异常在 `dump()` 中显示 `[TYPE:1]`（generic_error），丢失具体错误分类。

## 约束

### 仅冷路径使用

**类型**: 编码规范

**规则**: `deviant` 及其子类仅在启动阶段和致命错误路径使用。热路径（网络 I/O、协议解析、TLS 握手）禁止抛异常。

**违反后果**: 异常栈展开开销不可预测，破坏热路径延迟保证。异常对象构造涉及 `std::string` 堆分配。

**源码依据**: `deviant.hpp:7-8`

### 异常对象较大

**类型**: 资源上限

**规则**: `deviant` 包含 `std::error_code` + `std::source_location` + `std::runtime_error`（内含 `std::string`），总大小约 80-120 字节。

**违反后果**: 在内存紧张时频繁构造可能导致分配失败。

**源码依据**: `deviant.hpp:144-146`

## 子类实现模式

```cpp
class security : public deviant {
public:
    explicit security(fault::code err,
                      const std::source_location &loc = std::source_location::current())
        : deviant(fault::make_error_code(err), {}, loc) {}
protected:
    auto type_name() const noexcept -> std::string_view override { return "SECURITY"; }
};
```

## 引用关系

### 依赖

- [[core/fault/code|fault::code]]：错误码枚举
- [[core/fault/compatible|fault::compatible]]：`make_error_code()` 和 `category()`

### 被引用

- [[core/exception/network|network]]：继承
- [[core/exception/protocol|protocol]]：继承
- [[core/exception/security|security]]：继承
