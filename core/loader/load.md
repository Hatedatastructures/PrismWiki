---
tags: [loader, load]
layer: core
source: include/prism/loader/load.hpp
title: Loader Load
module: loader
updated: 2026-05-27
---

# Loader Load

配置加载适配器，将外部 JSON 配置转换为内部 `config` 结构，并将认证配置构建为运行时 `account::directory`。是外部世界与核心配置之间的防腐层。

## 核心接口

| 函数 | 说明 |
|------|------|
| `load(path)` | 加载 JSON 配置文件，返回 `config`。失败抛异常 |
| `build_dir(auth)` | 从认证配置构建 `shared_ptr<account::directory>` |

## 设计决策

### 为什么 load() 用异常而非错误码？

配置加载失败是致命错误，程序无法继续运行。异常自动展开调用栈，`main()` 的 `try/catch` 统一处理，不需要逐层检查错误码。这与热路径的双轨策略一致：启动路径用异常，运行路径用 `fault::code`。

**后果**: 调用方必须用 `try/catch` 包裹 `load()` 调用。

### 为什么用 `memory::string` 读取文件内容？

文件内容是一次性消费的临时字符串，用 PMR 分配避免全局堆锁。读取后通过 `string_view` 传递给反序列化器，零拷贝。

**后果**: `memory::string` 的分配器来自全局池，读取期间零堆分配。

### 为什么 password 要 SHA224 规范化？

代理协议（Trojan）传输的凭证是 SHA224 哈希值。`build_dir()` 将配置中的明文密码预先规范化为 SHA224，运行时认证时直接比较哈希值，避免每次连接都计算哈希。

**后果**: 目录中存储的是 SHA224 哈希字符串，非明文密码。

### 为什么 password 和 uuid 共享同一个 entry？

同一用户可能通过不同协议连接（Trojan 用 UUID，SOCKS5 用密码）。共享 entry 确保连接数限制跨协议生效，否则同一用户可绕过配额。

**后果**: entry 的 `shared_ptr` 被多个 map key 引用，直到所有 key 移除且所有 lease 释放才销毁。

## 约束

### load() 必须在 worker 创建前调用

**类型**: 调用顺序

**规则**: `loader::load()` 必须在创建 worker 线程池之前完成（`main.cpp` 启动流程保证）。

**违反后果**: worker 使用未初始化的配置，行为不可预测。

**源码依据**: `load.hpp:36-69`

### build_dir() 中的 reserve() 减少重新哈希

**类型**: 性能

**规则**: `build_dir()` 预估总条目数（每个用户的 password + uuid），先 `reserve()` 再插入。

**违反后果**: 不调用 `reserve()` 时每次插入可能触发 `unordered_map` 重新哈希，COW 复制开销增大。

**源码依据**: `load.hpp:85-97`

## 异常映射

| 阶段 | 异常类型 | 触发条件 |
|------|----------|----------|
| 文件打开 | `exception::security` | `ifstream::is_open()` 为 false |
| JSON 解析 | `exception::network(fault::code::parse_error)` | `deserialize()` 失败或抛 `std::exception` |
| JSON 解析 | `exception::network(fault::code::parse_error)` | 未知异常 |

## 引用关系

### 依赖

- [[core/transformer/overview|transformer]]：JSON 反序列化
- [[core/exception/overview|exception]]：启动阶段异常
- [[core/account/directory|account::directory]]：账户目录构建
- [[core/crypto/sha224|crypto::sha224]]：凭证规范化

### 被引用

- `main.cpp`：启动流程调用
