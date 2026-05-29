---
tags: [loader, overview]
layer: core
module: loader
source: include/prism/loader/load.hpp
title: Loader 模块
---

# Loader 模块

Loader 模块提供配置加载功能，将外部配置文件转换为内部配置结构。是外部 JSON 格式与内部类型之间的防腐层。

## 设计决策

### 为什么 loader 是防腐层？

外部配置文件格式（JSON 字段名、结构）和内部 `psm::config` 结构体是两个独立演化的东西。如果业务代码直接消费 JSON，格式变更会导致全项目改动。Loader 将 JSON 反序列化为中间结构，再转换为 `psm::config`，隔离格式变化。

**后果**: 配置格式变更只需修改 loader，不影响下游消费者。

### 为什么启动阶段用异常？

配置加载失败是致命错误，程序无法继续运行。异常自动展开调用栈，`main()` 的 `try/catch` 统一处理，不需要逐层检查错误码。

**后果**: Loader 抛出 `exception::security` 或 `exception::network`，调用方必须用 `try/catch`。

## 模块组成

| 组件 | 说明 | 源码 |
|------|------|------|
| [[core/loader/load]] | 配置加载器 | `prism/loader/load.hpp` |

## 核心接口

| 函数 | 说明 |
|------|------|
| `load(path)` | 加载配置文件，返回 `psm::config`。失败时抛异常 |
| `build_account_directory(auth)` | 从认证配置构建账户目录 |

## 约束

| 约束 | 规则 | 违反后果 | 来源 |
|------|------|----------|------|
| load() 在 worker 创建前调用 | main.cpp 启动流程保证顺序 | worker 使用未初始化配置，行为不可预测 | main.cpp |
| build_dir 无输入校验 | 假设 auth 结构已由 deserialize 验证 | 空 password/uuid 被跳过但不报错 | load.hpp:79 |
| 配置文件必须存在且可读 | 文件打开失败抛 exception::security | 进程终止 | load.hpp:40 |
| JSON 必须符合 psm::config 结构 | deserialize 失败抛 exception::network | 进程终止 | load.hpp:53-67 |
| build_dir 使用 local_pool | 账户目录分配器绑定当前线程 | 跨线程访问账户目录时分配器可能不匹配 | load.hpp:83 |

## 故障场景

### 1. 配置文件不存在

**触发条件**: `load(path)` 中 path 指向不存在的文件

**传播路径**: `std::ifstream::open` 失败 -> 抛出 `exception::security("system error : file open failed")`

**外部表现**: 进程终止，日志输出错误信息

### 2. JSON 格式错误

**触发条件**: 配置文件内容不是合法 JSON 或字段类型不匹配

**传播路径**: `glz::deserialize` 返回 false 或抛出 `std::exception` -> 捕获后抛出 `exception::network(fault::code::parse_error)`

**外部表现**: 进程终止，trace::error 输出解析错误详情

### 3. 必填字段缺失

**触发条件**: JSON 缺少必需的字段（如 `"agent"`）

**传播路径**: `deserialize` 生成的 config 结构中对应字段为零值 -> 下游模块使用零值配置

**外部表现**: 取决于下游模块如何处理零值（可能崩溃或静默错误）

### 4. 用户密码为空

**触发条件**: auth 用户列表中某个用户的 password 和 uuid 都为空

**传播路径**: `build_dir` 跳过该用户 -> 不插入 directory -> 该用户无法通过任何凭据认证

**外部表现**: 客户端使用该用户凭据连接时认证失败

## 跨模块契约

| 契约 | 方向 | 说明 |
|------|------|------|
| [[core/loader/config|config]] <- loader | 产出 | `load()` 返回填充好的 `psm::config` 结构体 |
| loader -> [[core/transformer/json|transformer]] | 依赖 | 调用 `json::deserialize()` 做 JSON 反序列化 |
| loader -> [[core/exception/overview|exception]] | 依赖 | 抛出 `exception::security` 和 `exception::network` |
| loader -> [[core/crypto/sha224|sha224]] | 依赖 | `build_dir()` 调用 `crypto::normalize_credential()` 哈希密码 |
| loader -> [[core/account/directory|account]] | 依赖 | `build_dir()` 构建账户目录 |
| loader -> [[core/trace/overview|trace]] | 依赖 | `trace::error()` 记录解析失败 |
| [[core/instance/overview|instance]] <- loader | 被依赖 | main.cpp 调用 load() 获取配置后创建 worker |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| config 结构体字段变更 | 所有消费者 | JSON 字段名/类型变更导致解析失败 |
| exception 类型变更 | main.cpp 的 try/catch | 捕获不到异常导致未处理终止 |
| normalize_credential 算法变更 | 所有用户密码 | 已存储的密码哈希失效，所有客户端认证失败 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| glaze 版本升级 | deserialize 行为 | 错误处理路径和返回值语义 |
| config 结构体新增字段 | load() 的 JSON 映射 | 新字段是否有默认值，是否必填 |
| account::directory 接口变更 | build_dir() 的插入逻辑 | 编译兼容性 |

## 引用关系

### 依赖

- [[core/loader/config|config]]：全局配置结构体
- [[core/transformer/json|json]]：JSON 反序列化
- [[core/exception/overview|exception]]：启动阶段异常
- [[core/crypto/sha224|sha224]]：密码哈希归一化
- [[core/account/directory|account]]：账户目录

### 被引用

- `main.cpp`：启动流程调用