# Wiki Schema

## Domain
Prism 知识库 — Prism 高性能代理引擎的项目文档，覆盖模块设计、实现细节、调试排障、客户端对接、Bug 记录等。

## 目录结构

```
wiki/
├── agent/              # 前端监听、会话管理、负载均衡、TLS
│   ├── overview.md     # 项目概览
│   ├── architecture.md # 分层架构设计
│   ├── modules.md      # 模块结构
│   ├── configuration.md # 配置详解
│   ├── testing.md      # 测试体系
│   ├── deployment.md   # 部署指南
│   └── troubleshooting.md # 故障排查
├── channel/            # 连接池、健康检查、传输层、加密传输
├── crypto/             # AEAD、Blake3、X25519、HKDF
├── multiplex/          # smux、yamux 实现细节
├── pipeline/           # 协议流水线（HTTP、SS、SOCKS5、Trojan、VLESS）
├── protocol/           # 协议格式、解析、中继
├── recognition/        # 协议识别、探测分析
├── resolve/            # DNS 解析、缓存、规则
├── stealth/            # Reality、ShadowTLS、Restls、ECH
├── memory/             # 内存池、容器
├── fault/              # 故障处理、错误码
├── client/             # mihomo 对接、配置模板、兼容性
├── bugs/               # Bug 记录（现象→排查→根因→修复）
├── dev/                # C++ 实现笔记、协程、构建系统
├── SCHEMA.md           # 本文件
├── index.md            # 分类索引
└── log.md              # 操作日志
```

## Conventions
- 文件名：小写，连字符，无空格（如 `smux-frame-format.md`）
- 每个页面以 YAML frontmatter 开头
- 使用 `[[wikilinks]]` 链接其他页面（每个页面 >= 3 个出站链接）
- 更新页面时更新 `updated` 日期
- 新页面必须添加到 `index.md`
- 每个操作必须记录到 `log.md`
- Bug 记录必须包含：现象、排查过程、根因、修复位置

## Frontmatter
```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: module | bug | config | dev | client
tags: [from taxonomy below]
related: [related modules or pages]
---
```

## Tag Taxonomy

### Prism 模块
- agent, channel, crypto, multiplex, pipeline, protocol, recognition, resolve, stealth, memory, fault

### 文档类型
- module, bug, config, dev, client, architecture, testing, deployment

### 协议相关
- socks5, http, shadowsocks, trojan, vless, tls, reality, shadowtls, restls, ech

### 实现细节
- smux, yamux, coroutine, memory-pool, connection-pool, dns, multiplexing

### 调试排障
- bug, troubleshooting, error-code, compatibility, regression

## Bug 记录格式
```yaml
---
title: Bug 标题
created: YYYY-MM-DD
type: bug
severity: critical | major | minor
modules: [affected modules]
status: fixed | investigating | workaround
fix_commit: commit hash (if applicable)
---
```

### Bug 正文结构
```
## 现象
描述用户可见的症状

## 排查过程
1. 第一步排查
2. 第二步排查
3. ...

## 根因
根本原因分析

## 修复
修复方案和代码位置

## 相关模块
[[module1]] [[module2]]

## 预防措施
如何避免类似问题
```

## Page Thresholds
- **创建页面**：当一个模块有 3+ 个文件或 500+ 行代码时
- **添加到现有页面**：当发现新的实现细节或 bug 时
- **不创建页面**：对于临时调试或一次性问题
- **拆分页面**：当页面超过 ~300 行时

## Update Policy
当发现新的 bug 或实现细节时：
1. 直接添加到对应模块页面或创建新 bug 页面
2. 更新相关模块的链接
3. 在 index.md 中添加新页面
4. 在 log.md 中记录操作

## API 函数文档标准

每个公开函数必须包含以下 7 个字段：

```markdown
### `函数名()`

> 源码: include/prism/module/file.hpp:XX | 实现: src/prism/module/file.cpp:XX

**功能**: 一句话描述函数的作用。

**签名**:
```cpp
auto function_name(参数列表) -> 返回类型;
```

**参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `param1` | `type` | 描述 |

**返回值**: `type` — 描述返回值含义和可能的错误码。

**调用（向下）**: `callee1()`, `callee2()` — 描述调用了哪些函数

**被调用（向上）**: `caller1()`, `caller2()` — 描述被哪些函数调用

**知识域**: `[[ref/...|知识域1]]`, `[[ref/...|知识域2]]`
```

### 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| 功能 | 是 | 一句话描述函数作用，不贴实现代码 |
| 参数/返回值 | 是 | 参数表格 + 返回值描述 |
| 调用（向下） | 是 | 该函数调用的其他函数，用 wikilink 链接 |
| 被调用（向上） | 是 | 调用该函数的其他函数/模块，用 wikilink 链接 |
| 涉及的知识域 | 是 | 相关的密码学/协议/网络知识域，链接到 ref/ 页面 |
| 源码位置 | 是 | 头文件路径:行号 + 实现文件路径:行号 |
| 交叉引用 | 是 | 至少 3 个出站 wikilink |

### 适用范围

此标准适用于以下模块的 API 文档：
- agent/、channel/、crypto/、pipeline/、protocol/、recognition/、resolve/、stealth/、multiplex/
- 根目录基础设施：memory.md、fault.md、exception.md、trace.md、transformer.md、loader.md、outbound.md

不适用于：client/、dev/、ref/、performance/、docs/

