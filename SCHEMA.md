---
title: 知识库规范
created: 2026-05-17
updated: 2026-05-25
layer: root
---

# Wiki Schema

## Domain
Prism 知识库 — Prism 高性能代理引擎的项目文档。

采用四层分离架构，各层职责明确，避免知识重复。

## 目录结构

```
wiki/
├── core/              # 模块实现细节（详细，开发排障）
│   ├── agent/         # 前端监听、会话管理、负载均衡
│   ├── channel/       # 连接池、传输层、Happy Eyeballs
│   ├── pipeline/      # 协议处理器、管道原语
│   ├── protocol/      # 协议格式、常量、配置
│   ├── stealth/       # TLS 伪装（Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel/ECH）
│   ├── recognition/   # 协议识别、探测分析
│   ├── resolve/       # DNS 解析、缓存、规则
│   ├── multiplex/     # Smux/Yamux 多路复用
│   ├── crypto/        # AEAD/HKDF/X25519/Blake3
│   ├── outbound/      # 出站代理、直连
│   ├── memory/        # PMR 内存池
│   ├── fault/         # 错误码
│   ├── exception/     # 异常体系
│   ├── trace/         # 日志系统
│   ├── transformer/   # JSON 序列化
│   ├── loader/        # 配置加载
│   ├── architecture.md    # 六阶段流水线架构
│   ├── startup.md         # 启动流程详解
│   ├── flow.md            # 协议处理流程
│   └── infrastructure.md  # 基础设施总览
│
├── dev/               # 开发规范、排障方法（详细）
│   ├── coding/        # 编码规范、协程约定、PMR 使用
│   ├── testing/       # 测试框架、基准测试、压力测试
│   ├── debugging/     # 排障方法、日志分析
│   ├── building/      # CMake 结构、构建命令
│   ├── performance/   # 性能优化、调优方法
│   ├── extending/     # 新协议/新伪装方案开发
│   ├── bugs/          # Bug 记录
│   └ roadmap.md       # 开发路线图
│
├── docs/              # 使用指南（简单）
│   ├── getting-started.md    # 快速开始
│   ├── deployment.md         # 部署指南
│   ├── configuration.md      # 配置说明（简化）
│   ├── client-setup.md       # 客户端配置
│   ├── faq.md                # 常见问题
│   ├── troubleshooting.md   # 故障排查（简化）
│   └ upgrade.md              # 升级指南
│   ├── security.md           # 安全注意事项
│   └ performance-tips.md     # 性能建议
│
├── ref/               # 参考知识（中等）
│   ├── mihomo/        # Mihomo 完整参考
│   │   ├── protocols/     # 所有出站协议（22个）
│   │   ├── transport/     # 传输层插件
│   │   ├── mux/           # 多路复用
│   │   ├── proxy-groups/  # 代理组
│   │   ├── rules/         # 规则系统
│   │   ├── dns/           # DNS 配置
│   │   ├── tun/           # TUN 模式
│   │   ├── config/        # 配置模板
│   │   ├── compatibility/ # Prism 兼容性
│   │   ├── implementation/# 实现参考
│   │   ├── sniffing/      # 协议嗅探
│   │   ├── listeners/     # 入站监听
│   │   ├── provider/      # Provider
│   │   ├── ntp/           # NTP 同步
│   │   └ experimental/    # 实验性功能
│   ├── crypto/        # 加密知识（原理）
│   ├── protocol/      # 协议知识（RFC）
│   ├── network/       # 网络知识
│   ├── programming/   # 编程知识（Boost.Asio/C++23协程）
│   └ glossary.md      # 术语表
│
├── skills/            # Claude Code Skills
│   └ prism-wiki-sync/ # 知识库同步检查
│
├── SCHEMA.md          # 本文件
├── index.md           # 总索引
└── log.md             # 操作日志
```

## 各层详细程度

### core/（详细层）

- **复杂函数**：逐行解释实现逻辑，包括：
  - 函数签名、参数、返回值
  - 每行代码的作用
  - 为什么这样实现（设计决策）
  - 调用链：调用者、被调用者（wikilink 标记）
  
- **简单函数**：只写用途，不逐行解释
  - getter/setter
  - 简单转换函数
  - 枚举定义

- **复杂判断标准**（综合）：
  - 代码行数 > 50
  - 逻辑分支 > 3
  - 涉及专业知识（加密/协议/协程）
  - 有复杂状态转换

- **调用链格式**：
  - 已标注位置：`调用 [[module/file|函数名]]()`
  - 未标注位置：`调用 [[module/file|函数名]]() — 源码: include/prism/module/file.hpp:45`

### dev/（详细层）

- 编码规范、协程约定等：详细说明每条规则及原因
- 排障方法：具体步骤、日志示例、常见错误
- Bug 记录：现象、排查过程、根因、修复位置

### docs/（简单层）

- 一句话说明功能
- 如何启用/配置
- 常见问题一句话解答

### ref/（中等层）

- 协议帧格式、字节布局
- 常量定义、枚举值含义
- 配置项说明
- 原理说明（不涉及实现细节）

## 知识不重复原则

同一知识点只在一个地方详细写，其他地方用 wikilink 引用：

- 函数实现细节 → core/
- 函数用途概述 → docs/ 或 ref/
- 相关原理 → ref/

示例：
- `core/stealth/reality/handshake.md` 详细写 Reality 握手实现
- `ref/mihomo/protocols/reality.md` 写 Reality 配置和兼容性，用 `详见 [[core/stealth/reality/handshake|Reality 握手实现]]` 引用

## Frontmatter

```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
layer: core | dev | docs | ref
source: 源码路径（core 层必填）
tags: [module tags]
---
```

## Tag Taxonomy

### 模块标签
- agent, channel, pipeline, protocol, stealth, recognition, resolve, multiplex, crypto, outbound, memory, fault, exception, trace, transformer, loader

### 协议标签
- socks5, http, trojan, vless, shadowsocks, reality, shadowtls, restls, anytls, trusttunnel, ech, hysteria2, tuic, wireguard, vmess

### 类型标签
- module, architecture, coding, testing, debugging, building, performance, bug, guide, reference

### 深层分析标签
- deep-dive — 深层故障分析
- failure-analysis — 故障分析
- known-issues — 已知问题
- resource-exhaustion — 资源耗尽
- misclassification — 误判
- implementation-boundary — 实现边界

## wikilink 规范

- 使用 Obsidian 格式：`[[path/to/page|显示名]]`
- 每个页面至少 3 个出站链接
- 链接到详细页面，不重复写内容

## 源码标注规范

core/ 层页面必须标注源码位置：

```
> 源码: include/prism/module/file.hpp:XX | 实现: src/prism/module/file.cpp:XX
```

函数说明中：

```
**调用（向下）**: `callee1()` — 源码: include/prism/module/file.hpp:45
**被调用（向上）**: [[caller/module|caller]] 的 `caller_func()`
```

## Bug 记录格式

Bug 记录放在 `dev/bugs/`：

```yaml
---
title: Bug 标题
created: YYYY-MM-DD
layer: dev
type: bug
severity: critical | major | minor
modules: [affected modules]
status: fixed | investigating | workaround
fix_commit: commit hash
---
```

正文结构：
- 现象
- 排查过程（具体步骤）
- 根因
- 修复位置
- 预防措施

deep-dive 类型正文结构：
1. 概述 — 问题描述和严重度
2. 详解 — 按故障场景分节，每节含触发条件、日志序列、影响
3. 排障方法 — 日志关键字、诊断命令
4. 相关文档 — wikilink 到 core 层实现细节

## Page Thresholds

- **创建页面**：模块有独立源码文件时
- **不创建页面**：临时调试、一次性问题
- **拆分页面**：页面超过 300 行时

## wiki-sync Skill

每次 push 时运行，检查知识库与源码同步状态。详见 `skills/prism-wiki-sync/SKILL.md`。