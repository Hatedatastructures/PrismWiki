---
title: "Loader 模块"
module: "loader"
type: module
tags: [loader, load, 配置, json, 账户目录, glaze, sha224]
created: 2026-05-15
updated: 2026-05-15
related:
  - transformer/json
  - agent/account/directory
  - agent/account/entry
  - crypto/sha224
  - trace/spdlog
  - exception/security
  - exception/network
---

# Loader 模块

> 模块: [[loader|loader]]

## 概述

Loader 模块负责配置文件加载和账户目录构建。充当外部世界与核心配置之间的防腐层，当前通过 glaze 库解析 JSON 格式。所有函数为 header-only 内联实现。

---

## load.hpp — 配置加载适配器

> 源码: `include/prism/loader/load.hpp`
> 模块: [[loader|loader]]

### 概述

配置加载适配器。负责将外部配置格式（JSON）转换为内部的 `psm::config` 结构，并将认证配置构建为运行时 `account::directory`。文件读取使用 `std::ifstream`，JSON 解析委托给 `transformer::json::deserialize()`。

### 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[transformer/json\|json]] | glaze JSON 反序列化 |
| 依赖 | [[agent/account/directory\|directory]] | 账户目录构建 |
| 依赖 | [[crypto/sha224\|sha224]] | 凭证规范化（normalize_credential） |
| 依赖 | [[exception/security\|security]] | 文件打开失败异常 |
| 依赖 | [[exception/network\|network]] | JSON 解析失败异常 |
| 依赖 | [[trace/spdlog\|trace]] | 解析错误日志记录 |
| 被依赖 | `main.cpp` | 启动时加载配置 |

### 命名空间

`psm::loader`

### 函数: load()

- **功能说明**: 从文件系统加载配置文件，解析 JSON 格式并转换为内部 `psm::config` 结构。流程：打开文件 -> 读取全部内容 -> 调用 `transformer::json::deserialize()` 解析 -> 返回 config 对象。解析失败时记录错误日志并抛出异常。
- **签名**: `inline auto load(std::string_view path) -> config`
- **参数**: `path` — 配置文件路径（支持相对/绝对路径）
- **返回值**: `config` — 转换后的配置对象
- **异常**: `exception::security` — 文件打开失败；`exception::network(fault::code::parse_error)` — JSON 解析失败
- **调用（向下）**: `std::ifstream` 文件读取、`transformer::json::deserialize()`、`trace::error()`、`exception::security` 构造、`exception::network` 构造
- **被调用（向上）**: `main.cpp` 启动流程（`psm::loader::load(configuration_path.string())`）
- **知识域**: JSON 配置文件加载

### 函数: build_account_directory()

- **功能说明**: 从认证配置构建账户目录。将配置中的统一用户注册到 `account::directory`。每个用户的 password 经 SHA224 规范化后注册，uuid 直接注册。两种凭证共享同一个 entry（共享连接数配额）。流程：预估条目数 -> reserve -> 遍历用户表 -> 创建 shared_entry -> 注册 password 和 uuid。
- **签名**: `inline auto build_account_directory(const agent::authentication &auth) -> std::shared_ptr<agent::account::directory>`
- **参数**: `auth` — 认证配置，包含统一用户表
- **返回值**: `std::shared_ptr<agent::account::directory>` — 共享的账户目录智能指针
- **调用（向下）**: `account::directory` 构造、`directory::reserve()`、`crypto::normalize_credential()`、`directory::insert()`、`account::entry` 构造、`memory::system::thread_local_pool()`
- **被调用（向上）**: `main.cpp` 启动流程（`psm::loader::build_account_directory(full_config.agent.auth)`）
- **知识域**: 账户目录构建

---

## loader 概述

Loader 模块的详细设计文档。两个核心函数：`load()` 负责 JSON 配置文件解析，`build_account_directory()` 负责认证配置到运行时账户目录的转换。两者均在 `main.cpp` 启动流程中顺序调用，是程序启动的关键路径。

## 相关页面

- [[transformer/json|json]] — JSON 序列化/反序列化，load() 的核心依赖
- [[agent/account/directory|directory]] — 账户目录，build_account_directory() 的输出
- [[agent/account/entry|entry]] — 账户条目，每个用户对应一个共享 entry
- [[crypto/sha224|sha224]] — SHA224 哈希，密码凭证规范化使用
- [[trace/spdlog|trace]] — 日志系统，load() 中记录解析错误
- [[exception/security|security]] — 安全异常，文件打开失败时抛出
- [[exception/network|network]] — 网络异常，JSON 解析失败时抛出
