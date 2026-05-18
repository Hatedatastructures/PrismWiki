---
title: 编码规范概览
description: Prism 项目的整体编码规范与约定，涵盖命名、协程、内存管理等方面
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