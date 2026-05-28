---
tags: [exception, overview]
layer: core
module: exception
source:
  - I:/code/Prism/include/prism/exception/deviant.hpp
  - I:/code/Prism/include/prism/exception/network.hpp
  - I:/code/Prism/include/prism/exception/protocol.hpp
  - I:/code/Prism/include/prism/exception/security.hpp
title: Exception 模块
---

# Exception 模块

结构化异常系统，基于 `std::error_code` 架构。仅用于启动阶段和致命错误，运行时热路径使用 `fault::code` 错误码。

## 设计决策

### 为什么异常基于 std::runtime_error 而非自定义根？

`std::runtime_error` 是标准库中"不可恢复运行时错误"的惯用基类。继承它允许异常被通用 `catch (const std::exception&)` 捕获，与第三方库（如 glaze JSON 解析）的异常统一处理。同时通过 `std::error_code` 成员保留结构化信息。

**后果**: `deviant` 对象较大（包含 `std::string` + `std::error_code` + `std::source_location`），不应在热路径构造。

### 为什么 deviant 是抽象基类？

强制每个异常子类提供 `type_name()`，使 `dump()` 输出包含分类标识（`SECURITY`/`NETWORK`/`PROTOCOL`），便于日志快速过滤。

**后果**: 不能直接 `throw deviant(...)`，必须使用具体子类。

### 为什么子类只有三个（network/protocol/security）？

三类异常覆盖所有冷路径错误场景：网络配置（network）、协议配置（protocol）、安全配置（security）。运行时的网络/协议/安全错误全部使用 `fault::code`，不需要对应的运行时异常类。

**后果**: 新增冷路径错误类型只需在三个子类中选择合适的即可，不需要创建新的异常类。

## 模块组成

| 组件 | 说明 | 源码 |
|------|------|------|
| [[core/exception/deviant\|deviant]] | 异常基类（抽象） | `prism/exception/deviant.hpp` |
| [[core/exception/network\|network]] | 网络配置异常 | `prism/exception/network.hpp` |
| [[core/exception/protocol\|protocol]] | 协议配置异常 | `prism/exception/protocol.hpp` |
| [[core/exception/security\|security]] | 安全配置异常 | `prism/exception/security.hpp` |

## 异常 vs 错误码决策

| 场景 | 使用 | 原因 |
|------|------|------|
| 配置文件解析失败 | `exception::security` | 启动阶段，致命 |
| TLS 证书加载失败 | `exception::security` | 启动阶段，致命 |
| 端口已被占用 | `exception::network` | 启动阶段，致命 |
| 协议配置不兼容 | `exception::protocol` | 启动阶段，致命 |
| 运行时网络 I/O 错误 | `fault::code` | 预期内，高频 |
| 运行时认证失败 | `fault::code` | 预期内，中频 |

## 相关模块

- [[core/fault/overview|Fault 模块]]：错误码系统（热路径）
- [[core/fault/compatible|fault::compatible]]：`make_error_code()` 用于异常构造
