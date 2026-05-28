---
title: Doxygen 注释规范
description: Prism 项目的 Doxygen 文档注释风格与标准
layer: dev
tags: [dev, coding, doxygen]
---

# Doxygen 注释规范

Prism 使用 Doxygen 风格的中文注释，确保代码文档化和可维护性。

## 注释风格

### 基本格式

使用 `@` 前缀的 Doxygen 标签：

```cpp
/**
 * @brief 简要描述
 * @details 详细描述（可选）
 *
 * @param name 参数说明
 * @return 返回值说明
 * @note 注意事项
 */
```

## 标签说明

### 文件注释

```cpp
/**
 * @file connection.hpp
 * @brief 连接管理模块
 * @details 定义连接池、连接抽象和连接生命周期管理
 */
#pragma once
```

### 类/结构体注释

```cpp
/**
 * @brief TCP 连接池
 * @details 管理连接的创建、复用和销毁，支持连接预热和健康检查
 *
 * 连接池特性：
 * - 线程安全的连接获取和释放
 * - 自动过期连接清理
 * - 连接预热支持
 */
class connection_pool {
public:
    // ...
};
```

### 函数注释

```cpp
/**
 * @brief 创建新连接
 * @details 异步建立 TCP 连接，支持超时控制和重试机制
 *
 * @param endpoint 目标端点
 * @param timeout 连接超时时间
 * @return 连接指针的 awaitable
 * @note 协程函数，需要在协程上下文中调用
 */
auto create_connection(
    const endpoint& endpoint,
    std::chrono::milliseconds timeout
) -> net::awaitable<connection_ptr>;
```

### 枚举注释

```cpp
/**
 * @brief 协议类型枚举
 * @details 定义支持的代理协议类型
 */
enum class protocol_type {
    http,        ///< HTTP 代理协议
    socks5,      ///< SOCKS5 代理协议
    trojan,      ///< Trojan 协议
    vless,       ///< VLESS 协议
    shadowsocks, ///< Shadowsocks 2022 协议
    unknown      ///< 未知协议
};
```

### 成员变量注释

```cpp
class session {
private:
    tcp::socket socket_;        ///< 客户端 socket
    endpoint remote_;           ///< 远程端点
    size_t bytes_received_{0};   ///< 已接收字节数
    bool closed_{false};        ///< 关闭标志
};
```

## 参考标准

参考 `channel/transport/reliable.hpp` 的注释风格：

```cpp
/**
 * @file reliable.hpp
 * @brief 可靠传输层实现
 * @details 提供 TCP 可靠传输抽象，支持 TLS 加密和多路复用
 */

/**
 * @brief 可靠传输通道
 * @details 封装 TCP socket 和可选 TLS 层，提供统一的读写接口
 *
 * 支持的传输模式：
 * - 纯 TCP
 * - TLS over TCP
 *
 * @note 线程安全：所有操作通过 strand 串行化
 */
class reliable_transport {
public:
    /**
     * @brief 异步读取数据
     * @param buffer 目标缓冲区
     * @return 读取字节数的 awaitable
     */
    auto read_some(buffer_span buffer) -> net::awaitable<size_t>;

    /**
     * @brief 异步写入数据
     * @param data 源数据
     * @return 写入字节数的 awaitable
     */
    auto write_some(const_buffer_span data) -> net::awaitable<size_t>;
};
```

## 注释最佳实践

### 1. 保持简洁

- `@brief` 用一句话概括功能
- `@details` 补充关键细节，避免重复

### 2. 说明协程函数

对于返回 `net::awaitable` 的函数，添加 `@note` 说明：

```cpp
/**
 * @brief 处理客户端会话
 * @return 无返回值的 awaitable
 * @note 协程函数，需要在 io_context 中执行
 */
auto handle_session() -> net::awaitable<void>;
```

### 3. 说明线程安全

```cpp
/**
 * @brief 获取统计信息
 * @return 当前统计快照
 * @note 线程安全，可从任意线程调用
 */
auto get_stats() const -> session_stats;
```

### 4. 说明生命周期

```cpp
/**
 * @brief 注册处理器
 * @param handler 处理器实例
 * @note handler 必须在程序生命周期内保持有效
 */
void register_handler(std::shared_ptr<handler> handler);
```

## 常用标签

| 标签 | 用途 | 示例 |
|------|------|------|
| `@file` | 文件说明 | `@file session.hpp` |
| `@brief` | 简要描述 | `@brief 会话管理器` |
| `@details` | 详细描述 | `@details 支持连接复用...` |
| `@param` | 参数说明 | `@param timeout 超时时间` |
| `@return` | 返回值说明 | `@return 连接指针` |
| `@note` | 注意事项 | `@note 线程安全` |
| `@see` | 参见 | `@see connection_pool` |
| `@todo` | 待办事项 | `@todo 支持连接预热` |

## 设计决策

### 为什么使用 `@` 前缀而非 `\`

Doxygen 支持两种命令前缀：`@` 和 `\`。Prism 选择 `@` 的原因：

1. **可读性**: `@brief`、`@param` 在代码中视觉上比 `\brief`、`\param` 更醒目，因为 `@` 在 C++ 代码中不是常见的转义字符
2. **避免转义混淆**: `\` 在 C++ 中是转义字符前缀（`\n`、`\t`），使用 `@` 避免在字符串和注释中使用相同前缀造成的视觉混淆
3. **Javadoc 惯例**: `@` 前缀源自 Javadoc，是文档工具中最广泛使用的风格，团队成员更容易适应
4. **一致性**: 项目内统一使用 `@` 前缀，不做混用

### 为什么使用中文注释

1. **团队语言**: Prism 开发团队以中文为主要交流语言，中文注释表达更精确、更高效
2. **概念精确**: 网络代理领域涉及大量协议概念（如"握手"、"探测"、"回落"），用中文表达比英文更准确，避免翻译歧义
3. **可维护性**: 注释的目标读者是项目维护者，用团队母语写注释降低理解成本
4. **代码与注释分离**: 标识符（类名、函数名）使用英文保证国际化兼容性，注释使用中文优化团队内部效率

### 为什么需要 Doxygen + Wiki 双重文档体系

1. **Doxygen 职责**: 紧贴代码的接口文档（参数、返回值、线程安全、生命周期），开发者修改代码时同步更新
2. **Wiki 职责**: 架构设计、模块间交互、设计决策、排障指南等高层知识，不适合放在代码注释中
3. **互补关系**: Doxygen 回答"这个函数做什么"，Wiki 回答"为什么这样设计"和"模块如何协作"
4. **知识沉淀**: 代码注释随代码变更而更新，Wiki 文档记录设计演进的决策过程，防止知识流失

## 相关文档

- [[overview]] - 编码规范概览
- [[naming]] - 命名规范
- [[dev/modules|modules]] - 模块结构