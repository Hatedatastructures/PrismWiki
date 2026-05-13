---
title: Loader 模块概述
created: 2026-05-13
updated: 2026-05-13
type: module
tags: [loader, configuration, overview]
related: [[transformer], [memory], [agent/overview], [architecture]]
---

# Loader 模块概述

## 1. 模块定位

Loader 模块是 Prism 的**配置加载适配器**，负责将外部 JSON 配置文件转换为内部 `psm::config` 结构，并将认证配置构建为运行时 `account::directory`。

**核心价值**：充当外部世界与核心配置之间的防腐层，隔离 JSON 解析细节。

## 2. 核心功能

### 2.1 配置加载

`loader::load(path)` 从文件系统加载 JSON 配置文件并转换为 `psm::config`：

```cpp
inline auto load(std::string_view path) -> config;
```

**流程**：
1. 以二进制模式打开配置文件
2. 读取全部内容到 `memory::string`
3. 调用 `transformer::json::deserialize()` 解析 JSON
4. 成功返回 `config` 对象，失败抛出异常

**错误处理**：
- 文件打开失败 → 抛出 `exception::security`
- JSON 解析失败 → 记录错误日志，抛出 `exception::network(parse_error)`

### 2.2 账户目录构建

`loader::build_account_directory(auth)` 从认证配置构建运行时账户目录：

```cpp
inline auto build_account_directory(const agent::authentication &auth)
    -> std::shared_ptr<agent::account::directory>;
```

**流程**：
1. 创建 `account::directory`（使用线程局部池）
2. 预估条目数并 `reserve`
3. 遍历用户列表，每个用户的 `password` 和 `uuid` 注册到目录
4. `password` 经 `crypto::normalize_credential()` 规范化（SHA-224）
5. 同一用户的 `password` 和 `uuid` 共享同一个 `entry`（共享连接数配额）

## 3. 设计原则

1. **防腐层**：隔离外部 JSON 格式与内部 C++ 结构
2. **快速失败**：配置错误在启动阶段立即报告，不在运行时暴露
3. **header-only**：所有实现在头文件中，无源文件
4. **异常传播**：加载失败抛出异常（启动阶段允许异常）

## 4. 与其他模块的关系

- **[[transformer]]**：调用 `deserialize()` 解析 JSON
- **[[memory]]**：使用 `memory::string` 和 `memory::system::thread_local_pool()`
- **[[agent]]**：构建 `account::directory` 传递给 Agent
- **[[crypto]]**：使用 `crypto::normalize_credential()` 规范化密码
- **[[exception]]**：加载失败抛出 `security` / `network` 异常
- **[[trace]]**：解析错误通过 `trace::error()` 记录日志

## 5. 配置文件搜索顺序

1. **命令行参数**指定的路径
2. **可执行文件同目录**下的 `configuration.json`

## 6. 典型使用场景

### 启动流程中的位置

```cpp
int main() {
    memory::system::enable_global_pooling();
    stealth::register_all_schemes();
    auto cfg = loader::load(config_path);          // 配置加载
    trace::init(cfg.trace);                        // 日志初始化
    auto dir = loader::build_account_directory(cfg.auth);  // 账户目录
    // ... 创建 worker、listener
}
```

## 相关页面

- [[transformer]] — Transformer 模块
- [[memory]] — Memory 模块
- [[agent]] — Agent 模块
- [[architecture]] — 架构设计
