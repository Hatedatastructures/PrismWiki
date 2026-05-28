---
layer: dev
title: Prism 扩展开发概览
tags: [dev, extending, overview]
---

# Prism 扩展开发概览

Prism 采用模块化架构，支持多种扩展方式：新协议接入、新伪装方案、新传输层、新功能模块。本页概述扩展架构与开发流程。

## 扩展架构概览

```
Session 层:  session::start() → probe → detect() → protocol_type
Dispatch 层: registry::create(type) → handler 实例
Pipeline 层: handler::process() → parse → auth → resolve → dial → tunnel
Stealth 层:  scheme_registry → scheme_executor → 方案匹配
Transport 层: transmission 接口 → reliable/websocket/...
```

## 扩展类型

### 1. 新代理协议

在 dispatch 层添加新的代理协议 handler：

- 定义 `protocol_type` 枚举值
- 实现协议检测逻辑（首包嗅探）
- 实现 pipeline 处理流程
- 注册 handler 到工厂

详细指南: [[extending/protocol]]

### 2. 新伪装方案

在 stealth 层添加新的 TLS 伪装方案：

- 定义 `scheme` 类实现 `execute()` 方法
- 注册到 `scheme_registry`
- 配置加载与验证

详细指南: [[extending/stealth]]

### 3. 新传输层

在 channel/transport 层添加新的传输实现：

- 继承 `transmission` 基类
- 实现 `async_read_some` / `async_write_some`
- 处理帧格式与协议握手

详细指南: [[extending/module]]

### 4. 新功能模块

添加独立功能模块：

- DNS 解析器扩展
- 路由规则引擎
- 监控指标系统

详细指南: [[extending/module]]

## 核心接口

### Handler 接口

```cpp
// include/prism/instance/dispatch/handler.hpp
class handler : public std::enable_shared_from_this<handler>
{
public:
    virtual ~handler() = default;
    
    // 主处理流程
    virtual auto process(session_context ctx) -> net::awaitable<void> = 0;
    
    // 协议类型标识
    virtual auto type() const -> protocol_type = 0;
    
    // 协议名称（用于日志）
    virtual auto name() const -> std::string_view = 0;
};
```

### Scheme 接口

```cpp
// include/prism/stealth/registry.hpp
struct scheme_result {
    protocol::protocol_type detected;
    shared_transmission transport;
    fault::code error;
};

class scheme {
public:
    virtual auto execute(scheme_context ctx) 
        -> net::awaitable<scheme_result> = 0;
    virtual auto is_enabled() const -> bool = 0;
};
```

### Transmission 接口

```cpp
// include/prism/transport/transmission.hpp
class transmission {
public:
    virtual auto async_read_some(std::span<std::byte> buffer)
        -> net::awaitable<std::size_t> = 0;
    virtual auto async_write_some(std::span<std::byte const> buffer)
        -> net::awaitable<std::size_t> = 0;
    virtual auto close() -> net::awaitable<void> = 0;
    virtual auto cancel() -> void = 0;
};
```

## 开发流程

### 通用步骤

1. **设计阶段**: 分析需求，确定扩展点
2. **接口定义**: 定义新类继承基类接口
3. **实现逻辑**: 编写核心处理逻辑
4. **注册集成**: 注册到工厂/注册表
5. **配置加载**: 添加配置字段与解析
6. **测试编写**: 单元测试 + 集成测试
7. **文档更新**: 更新相关文档

### 协程规范

扩展开发必须遵循 Prism 协程规范：

1. **禁止阻塞调用**: 所有 I/O 使用 `co_await`
2. **生命周期管理**: `co_spawn` lambda 捕获 `self`
3. **PMR 内存**: 热路径使用 `memory::vector<T>`
4. **错误处理**: 热路径用 `fault::code`

## 扩展子页面

- [[extending/protocol]] — 新协议接入指南
- [[extending/stealth]] — 新伪装方案开发
- [[extending/module]] — 新模块开发

## 相关架构

- [[dispatch/table]] — Handler 注册表
- [[dev/modules|modules]] — 模块系统架构
- [[ref/programming/c++23-coroutines|cpp23-coroutines]] — 协程编程规范
- [[ref/programming/pmr-concepts|pmr-memory-pool]] — PMR 内存管理