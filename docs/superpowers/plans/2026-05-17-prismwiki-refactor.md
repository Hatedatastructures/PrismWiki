# PrismWiki 知识库重构实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 重构 PrismWiki 知识库，建立 core/dev/docs/ref 四层结构，编写详细模块文档，实现 wiki-sync skill。

**Architecture:** 四层分离：core（模块实现细节）、dev（开发规范/排障）、docs（用户指南）、ref（参考知识+完整 mihomo）。wiki-sync skill 替换原有 bug-to-wiki/module-analysis。知识不重复原则：同一知识点只在一处详细写，其他用 wikilink 引用。

**Tech Stack:** Markdown, Claude Code Skills, wikilink 引用系统

---

## 执行约束

- **并行上限**: 最多 3 个 agent 同时运行
- **依赖考虑**: 目录结构创建完成后才能写内容；overview 页面先创建才能被其他页面引用
- **源码位置**: 
  - Prism: `I:/code/Prism/`
  - Mihomo: `I:/code/Prism/logs/mihomo-Meta/`
- **详细程度**: core/dev 详细，ref 中等，docs 简单
- **不重复原则**: 同一知识点只在一处详细写

---

## 阶段 1：创建目录结构（并行 3 agent）

### Task 1: 创建 core/ 目录结构

**Files:**
- Create: `H:/PrismWiki/core/` 及所有子目录

- [ ] **Step 1: 创建 core/agent 子目录**

```bash
mkdir -p "H:/PrismWiki/core/agent/front"
mkdir -p "H:/PrismWiki/core/agent/session"
mkdir -p "H:/PrismWiki/core/agent/worker"
mkdir -p "H:/PrismWiki/core/agent/account"
mkdir -p "H:/PrismWiki/core/agent/dispatch"
```

- [ ] **Step 2: 创建 core/channel 子目录**

```bash
mkdir -p "H:/PrismWiki/core/channel/transport"
mkdir -p "H:/PrismWiki/core/channel/connection"
mkdir -p "H:/PrismWiki/core/channel/eyeball"
mkdir -p "H:/PrismWiki/core/channel/adapter"
```

- [ ] **Step 3: 创建 core/pipeline 和 core/protocol 子目录**

```bash
mkdir -p "H:/PrismWiki/core/pipeline/protocols"
mkdir -p "H:/PrismWiki/core/protocol/common"
mkdir -p "H:/PrismWiki/core/protocol/http"
mkdir -p "H:/PrismWiki/core/protocol/socks5"
mkdir -p "H:/PrismWiki/core/protocol/trojan"
mkdir -p "H:/PrismWiki/core/protocol/vless"
mkdir -p "H:/PrismWiki/core/protocol/shadowsocks"
mkdir -p "H:/PrismWiki/core/protocol/tls"
```

- [ ] **Step 4: 创建 core/stealth 子目录**

```bash
mkdir -p "H:/PrismWiki/core/stealth/reality"
mkdir -p "H:/PrismWiki/core/stealth/shadowtls"
mkdir -p "H:/PrismWiki/core/stealth/restls"
mkdir -p "H:/PrismWiki/core/stealth/anytls"
mkdir -p "H:/PrismWiki/core/stealth/trusttunnel"
mkdir -p "H:/PrismWiki/core/stealth/ech"
```

- [ ] **Step 5: 创建 core 其他模块子目录**

```bash
mkdir -p "H:/PrismWiki/core/recognition/probe"
mkdir -p "H:/PrismWiki/core/resolve/dns/detail"
mkdir -p "H:/PrismWiki/core/multiplex/smux"
mkdir -p "H:/PrismWiki/core/multiplex/yamux"
mkdir -p "H:/PrismWiki/core/crypto"
mkdir -p "H:/PrismWiki/core/outbound"
mkdir -p "H:/PrismWiki/core/memory"
mkdir -p "H:/PrismWiki/core/fault"
mkdir -p "H:/PrismWiki/core/exception"
mkdir -p "H:/PrismWiki/core/trace"
mkdir -p "H:/PrismWiki/core/transformer"
mkdir -p "H:/PrismWiki/core/loader"
```

- [ ] **Step 6: 验证 core/ 目录创建成功**

```bash
ls -la "H:/PrismWiki/core/"
```
Expected: 看到 agent/, channel/, pipeline/, protocol/, stealth/, recognition/, resolve/, multiplex/, crypto/, outbound/, memory/, fault/, exception/, trace/, transformer/, loader/ 目录

### Task 2: 创建 dev/ 目录结构

**Files:**
- Create: `H:/PrismWiki/dev/` 及所有子目录

- [ ] **Step 1: 创建 dev 子目录**

```bash
mkdir -p "H:/PrismWiki/dev/coding"
mkdir -p "H:/PrismWiki/dev/testing"
mkdir -p "H:/PrismWiki/dev/debugging"
mkdir -p "H:/PrismWiki/dev/building"
mkdir -p "H:/PrismWiki/dev/performance"
mkdir -p "H:/PrismWiki/dev/extending"
mkdir -p "H:/PrismWiki/dev/bugs"
```

- [ ] **Step 2: 验证 dev/ 目录创建成功**

```bash
ls -la "H:/PrismWiki/dev/"
```
Expected: 看到 coding/, testing/, debugging/, building/, performance/, extending/, bugs/ 目录

### Task 3: 创建 docs/, ref/, skills/ 目录结构

**Files:**
- Create: `H:/PrismWiki/docs/`, `H:/PrismWiki/ref/`, `H:/PrismWiki/skills/`

- [ ] **Step 1: 创建 docs 目录**

```bash
mkdir -p "H:/PrismWiki/docs"
```

- [ ] **Step 2: 创建 ref/mihomo 子目录**

```bash
mkdir -p "H:/PrismWiki/ref/mihomo/protocols"
mkdir -p "H:/PrismWiki/ref/mihomo/transport"
mkdir -p "H:/PrismWiki/ref/mihomo/mux"
mkdir -p "H:/PrismWiki/ref/mihomo/proxy-groups"
mkdir -p "H:/PrismWiki/ref/mihomo/rules"
mkdir -p "H:/PrismWiki/ref/mihomo/dns"
mkdir -p "H:/PrismWiki/ref/mihomo/tun"
mkdir -p "H:/PrismWiki/ref/mihomo/config"
mkdir -p "H:/PrismWiki/ref/mihomo/compatibility"
mkdir -p "H:/PrismWiki/ref/mihomo/implementation"
mkdir -p "H:/PrismWiki/ref/mihomo/sniffing"
mkdir -p "H:/PrismWiki/ref/mihomo/script"
mkdir -p "H:/PrismWiki/ref/mihomo/listeners"
mkdir -p "H:/PrismWiki/ref/mihomo/provider"
mkdir -p "H:/PrismWiki/ref/mihomo/ntp"
mkdir -p "H:/PrismWiki/ref/mihomo/experimental"
```

- [ ] **Step 3: 创建 ref 其他子目录**

```bash
mkdir -p "H:/PrismWiki/ref/crypto"
mkdir -p "H:/PrismWiki/ref/protocol"
mkdir -p "H:/PrismWiki/ref/network"
mkdir -p "H:/PrismWiki/ref/programming"
```

- [ ] **Step 4: 创建 skills 子目录**

```bash
mkdir -p "H:/PrismWiki/skills/prism-wiki-sync/references"
```

- [ ] **Step 5: 验证目录创建成功**

```bash
ls -la "H:/PrismWiki/docs"
ls -la "H:/PrismWiki/ref/"
ls -la "H:/PrismWiki/ref/mihomo/"
ls -la "H:/PrismWiki/skills/"
```
Expected: 所有目录存在

---

## 阶段 2：重写基础规范文件（并行 3 agent）

### Task 4: 重写 SCHEMA.md

**Files:**
- Modify: `H:/PrismWiki/SCHEMA.md`

- [ ] **Step 1: 读取现有 SCHEMA.md 作为参考**

```bash
cat "H:/PrismWiki/SCHEMA.md" | head -50
```
Expected: 现有规范内容

- [ ] **Step 2: 写入新的 SCHEMA.md（第一部分：Domain 和目录结构）**

内容见下方完整代码块，由于内容很长，使用 Write 工具写入完整文件。

- [ ] **Step 3: 验证 SCHEMA.md 写入成功**

```bash
head -30 "H:/PrismWiki/SCHEMA.md"
```
Expected: 新的规范内容开头

### Task 5: 重写 index.md

**Files:**
- Modify: `H:/PrismWiki/index.md`

- [ ] **Step 1: 读取现有 index.md 作为参考**

```bash
head -100 "H:/PrismWiki/index.md"
```
Expected: 现有索引内容

- [ ] **Step 2: 写入新的 index.md**

内容见下方完整代码块。

- [ ] **Step 3: 验证 index.md 写入成功**

```bash
head -50 "H:/PrismWiki/index.md"
```
Expected: 新的索引内容

### Task 6: 创建 core/overview.md

**Files:**
- Create: `H:/PrismWiki/core/overview.md`

- [ ] **Step 1: 写入 core/overview.md**

```markdown
---
title: Core 层总览
created: 2026-05-17
layer: core
tags: [architecture, overview]
---

# Core 层总览

Core 层是知识库最详细的层级，包含 Prism 所有模块的实现细节。

## 职责

- 函数实现详解（复杂函数逐行解释）
- 调用链关系
- 状态变化说明
- 错误处理逻辑

## 详细程度

- **复杂函数**: 逐行解释，说明每行代码作用、设计决策
- **简单函数**: 只写用途，不逐行解释
- **枚举/常量**: 列出值和含义，不逐行解释

复杂判断标准（综合）：
1. 代码行数 > 50
2. 逻辑分支 > 3
3. 涉及专业知识（加密/协议/协程）
4. 有复杂状态转换

## 模块列表

| 模块 | 职责 | 源码位置 |
|------|------|----------|
| [[agent/overview|Agent]] | 前端监听、会话管理 | `include/prism/agent/` |
| [[channel/overview|Channel]] | 连接池、传输层 | `include/prism/channel/` |
| [[pipeline/overview|Pipeline]] | 协议处理器 | `include/prism/pipeline/` |
| [[protocol/overview|Protocol]] | 协议格式、常量 | `include/prism/protocol/` |
| [[stealth/overview|Stealth]] | TLS 伪装 | `include/prism/stealth/` |
| [[recognition/overview|Recognition]] | 协议识别 | `include/prism/recognition/` |
| [[resolve/overview|Resolve]] | DNS 解析 | `include/prism/resolve/` |
| [[multiplex/overview|Multiplex]] | 多路复用 | `include/prism/multiplex/` |
| [[crypto/overview|Crypto]] | 加密模块 | `include/prism/crypto/` |
| [[outbound/overview|Outbound]] | 出站代理 | `include/prism/outbound/` |
| [[memory/overview|Memory]] | PMR 内存池 | `include/prism/memory/` |
| [[fault/overview|Fault]] | 错误码 | `include/prism/fault/` |
| [[exception/overview|Exception]] | 异常体系 | `include/prism/exception/` |
| [[trace/overview|Trace]] | 日志系统 | `include/prism/trace/` |
| [[transformer/overview|Transformer]] | JSON 序列化 | `include/prism/transformer/` |
| [[loader/overview|Loader]] | 配置加载 | `include/prism/loader/` |

## 架构总览

详见 [[architecture|六阶段流水线架构]]

## 启动流程

详见 [[startup|启动流程详解]]

## 协议处理流程

详见 [[flow|协议处理流程详解]]

## 基础设施

详见 [[infrastructure|基础设施总览]]
```

---

## 阶段 3：创建核心架构页面（并行 3 agent）

### Task 7: 创建 core/architecture.md

**Files:**
- Create: `H:/PrismWiki/core/architecture.md`
- Source: `I:/code/Prism/CLAUDE.md` §架构概览

- [ ] **Step 1: 读取 CLAUDE.md 架构概览部分**

```bash
grep -A 50 "架构概览" "I:/code/Prism/CLAUDE.md"
```
Expected: 六阶段流水线架构说明

- [ ] **Step 2: 写入 core/architecture.md**

内容包含：
1. 六阶段流水线 ASCII 图
2. 每阶段职责说明
3. 各阶段对应模块 wikilink
4. 分层架构说明（Front → Worker → Session → Recognition → Dispatch → Pipeline → Resolve → Multiplex → Channel → Stealth）

完整内容见下方。

### Task 8: 创建 core/startup.md

**Files:**
- Create: `H:/PrismWiki/core/startup.md`
- Source: `I:/code/Prism/CLAUDE.md` §启动流程, `I:/code/Prism/src/main.cpp`

- [ ] **Step 1: 读取 CLAUDE.md 启动流程部分**

```bash
grep -A 20 "启动流程" "I:/code/Prism/CLAUDE.md"
```
Expected: 9 步启动顺序

- [ ] **Step 2: 写入 core/startup.md**

内容包含：
1. main.cpp 启动顺序 9 步（每步用 wikilink 标记函数）
2. 配置文件查找顺序
3. 注意事项

### Task 9: 创建 core/flow.md

**Files:**
- Create: `H:/PrismWiki/core/flow.md`
- Source: `I:/code/Prism/CLAUDE.md` §协议处理流程

- [ ] **Step 1: 读取 CLAUDE.md 协议处理流程部分**

```bash
grep -A 30 "协议处理流程" "I:/code/Prism/CLAUDE.md"
```
Expected: 协议处理流程说明

- [ ] **Step 2: 写入 core/flow.md**

内容包含：
1. 协议处理流程 ASCII 图
2. Recognition 三阶段流水线
3. 每步骤用 wikilink 标记

---

## 阶段 4：创建 dev/ 基础页面（并行 3 agent）

### Task 10: 创建 dev/overview.md

**Files:**
- Create: `H:/PrismWiki/dev/overview.md`

- [ ] **Step 1: 写入 dev/overview.md**

```markdown
---
title: Dev 层总览
created: 2026-05-17
layer: dev
tags: [dev, overview]
---

# Dev 层总览

Dev 层包含开发规范、排障方法、Bug 记录等开发相关内容。

## 职责

- 编码规范详解
- 测试体系说明
- 排障方法步骤
- 构建系统说明
- 性能优化指南
- 扩展开发指南
- Bug 记录归档

## 详细程度

与 core/ 层相同，详细说明每条规范、每个步骤的原因和具体操作。

## 子目录

| 目录 | 职责 |
|------|------|
| [[coding/overview|coding]] | 编码规范、协程约定、PMR 使用 |
| [[testing/overview|testing]] | 测试框架、命令、基准测试 |
| [[debugging/overview|debugging]] | 排障方法、日志分析 |
| [[building/overview|building]] | CMake 结构、构建命令 |
| [[performance/overview|performance]] | 性能优化、调优方法 |
| [[extending/overview|extending]] | 新协议/新伪装方案开发 |
| [[bugs/template|bugs]] | Bug 记录模板和归档 |

## 开发路线图

详见 [[roadmap|开发路线图]]
```

### Task 11: 创建 dev/coding/ 页面

**Files:**
- Create: `H:/PrismWiki/dev/coding/overview.md`
- Create: `H:/PrismWiki/dev/coding/naming.md`
- Create: `H:/PrismWiki/dev/coding/coroutine.md`
- Create: `H:/PrismWiki/dev/coding/pmr.md`
- Create: `H:/PrismWiki/dev/coding/doxygen.md`
- Create: `H:/PrismWiki/dev/coding/lifecycle.md`
- Create: `H:/PrismWiki/dev/coding/error.md`
- Source: `I:/code/Prism/CLAUDE.md` §命名与编码规范, §协程约定, §PMR内存策略, §生命周期安全, §错误处理

- [ ] **Step 1: 读取 CLAUDE.md 相关部分**

读取命名规范、协程约定、PMR、生命周期、错误处理部分，作为内容来源。

- [ ] **Step 2: 写入 dev/coding/overview.md**

- [ ] **Step 3: 写入 dev/coding/naming.md** — 基于 CLAUDE.md §命名与编码规范

- [ ] **Step 4: 写入 dev/coding/coroutine.md** — 基于 CLAUDE.md §协程约定、§协程纯度要求（包含禁止表）

- [ ] **Step 5: 写入 dev/coding/pmr.md** — 基于 CLAUDE.md §PMR内存策略

- [ ] **Step 6: 写入 dev/coding/doxygen.md** — 基于 CLAUDE.md §命名与编码规范-注释

- [ ] **Step 7: 写入 dev/coding/lifecycle.md** — 基于 CLAUDE.md §生命周期安全

- [ ] **Step 8: 写入 dev/coding/error.md** — 基于 CLAUDE.md §错误处理（双轨策略）

### Task 12: 创建 dev/building/ 页面

**Files:**
- Create: `H:/PrismWiki/dev/building/overview.md`
- Create: `H:/PrismWiki/dev/building/cmake.md`
- Create: `H:/PrismWiki/dev/building/dependencies.md`
- Create: `H:/PrismWiki/dev/building/commands.md`
- Create: `H:/PrismWiki/dev/building/options.md`
- Source: `I:/code/Prism/CLAUDE.md` §构建命令, §依赖项, §构建系统结构

- [ ] **Step 1: 读取 CLAUDE.md 构建相关部分**

- [ ] **Step 2: 写入 dev/building/overview.md**

- [ ] **Step 3: 写入 dev/building/cmake.md** — CMake 结构详解

- [ ] **Step 4: 写入 dev/building/dependencies.md** — 依赖管理

- [ ] **Step 5: 写入 dev/building/commands.md** — 构建命令（Release/Debug/测试）

- [ ] **Step 6: 写入 dev/building/options.md** — 构建选项

---

## 阶段 5：创建 docs/ 用户指南页面（并行 1 agent）

### Task 13: 创建 docs/ 所有页面

**Files:**
- Create: `H:/PrismWiki/docs/overview.md`
- Create: `H:/PrismWiki/docs/getting-started.md`
- Create: `H:/PrismWiki/docs/deployment.md`
- Create: `H:/PrismWiki/docs/configuration.md`
- Create: `H:/PrismWiki/docs/client-setup.md`
- Create: `H:/PrismWiki/docs/faq.md`
- Create: `H:/PrismWiki/docs/troubleshooting.md`
- Create: `H:/PrismWiki/docs/upgrade.md`
- Create: `H:/PrismWiki/docs/security.md`
- Create: `H:/PrismWiki/docs/performance-tips.md`
- Source: `I:/code/Prism/docs/tutorial/`, `I:/code/Prism/src/configuration.json`

- [ ] **Step 1: 读取 docs/tutorial/ 作为参考**

```bash
ls "I:/code/Prism/docs/tutorial/"
```
Expected: getting-started.md, configuration.md, deployment.md, troubleshooting.md, faq.md

- [ ] **Step 2: 写入 docs/overview.md**

- [ ] **Step 3: 写入 docs/getting-started.md** — 快速开始

- [ ] **Step 4: 写入 docs/deployment.md** — 部署指南

- [ ] **Step 5: 写入 docs/configuration.md** — 配置说明（简化版）

- [ ] **Step 6: 写入 docs/client-setup.md** — 客户端配置

- [ ] **Step 7: 写入 docs/faq.md** — 常见问题

- [ ] **Step 8: 写入 docs/troubleshooting.md** — 故障排查（简化版）

- [ ] **Step 9: 写入 docs/upgrade.md** — 升级指南

- [ ] **Step 10: 写入 docs/security.md** — 安全注意事项

- [ ] **Step 11: 写入 docs/performance-tips.md** — 性能建议

---

## 阶段 6：创建 core/agent/ 模块页面（并行 3 agent）

### Task 14: 创建 core/agent/overview.md 和核心页面

**Files:**
- Create: `H:/PrismWiki/core/agent/overview.md`
- Create: `H:/PrismWiki/core/agent/config.md`
- Create: `H:/PrismWiki/core/agent/context.md`
- Source: `I:/code/Prism/include/prism/agent/`, `I:/code/Prism/CLAUDE.md` §Agent模块结构

- [ ] **Step 1: 读取 agent 模块源码结构**

```bash
ls "I:/code/Prism/include/prism/agent/"
```
Expected: account/, dispatch/, front/, session/, worker/, context.hpp, config.hpp

- [ ] **Step 2: 写入 core/agent/overview.md**

- [ ] **Step 3: 写入 core/agent/config.md** — 源码: include/prism/agent/config.hpp

- [ ] **Step 4: 写入 core/agent/context.md** — 源码: include/prism/agent/context.hpp

### Task 15: 创建 core/agent/front/ 页面

**Files:**
- Create: `H:/PrismWiki/core/agent/front/listener.md`
- Create: `H:/PrismWiki/core/agent/front/balancer.md`
- Source: `I:/code/Prism/include/prism/agent/front/listener.hpp`, `I:/code/Prism/include/prism/agent/front/balancer.hpp`

- [ ] **Step 1: 读取 listener.hpp 源码**

```bash
cat "I:/code/Prism/include/prism/agent/front/listener.hpp"
```
Expected: listener 类定义

- [ ] **Step 2: 读取 balancer.hpp 源码**

```bash
cat "I:/code/Prism/include/prism/agent/front/balancer.hpp"
```
Expected: balancer 类定义

- [ ] **Step 3: 写入 core/agent/front/listener.md**

内容要点：
- 源码位置标注
- 功能说明：监听端口，接受连接，计算亲和性哈希
- 调用链：调用 [[agent/front/balancer]] 的 select()
- 被调用：main.cpp 启动

- [ ] **Step 4: 写入 core/agent/front/balancer.md**

内容要点：
- 源码位置标注
- 功能说明：选择 worker 分发连接
- 调用链：调用 [[agent/worker/worker]] 的 post()
- 被调用：[[agent/front/listener]]

### Task 16: 创建 core/agent/session/ 和 worker/ 页面

**Files:**
- Create: `H:/PrismWiki/core/agent/session/session.md`
- Create: `H:/PrismWiki/core/agent/worker/worker.md`
- Create: `H:/PrismWiki/core/agent/worker/launch.md`
- Create: `H:/PrismWiki/core/agent/worker/stats.md`
- Create: `H:/PrismWiki/core/agent/worker/tls.md`
- Source: `I:/code/Prism/include/prism/agent/session/session.hpp`, `I:/code/Prism/include/prism/agent/worker/`

- [ ] **Step 1: 读取相关源码**

- [ ] **Step 2: 写入 core/agent/session/session.md**

- [ ] **Step 3: 写入 core/agent/worker/worker.md**

- [ ] **Step 4: 写入 core/agent/worker/launch.md**

- [ ] **Step 5: 写入 core/agent/worker/stats.md**

- [ ] **Step 6: 写入 core/agent/worker/tls.md**

---

## 阶段 7：创建 core/channel/ 模块页面（并行 3 agent）

### Task 17: 创建 core/channel/overview.md 和核心页面

**Files:**
- Create: `H:/PrismWiki/core/channel/overview.md`
- Create: `H:/PrismWiki/core/channel/health.md`
- Source: `I:/code/Prism/include/prism/channel/`

- [ ] **Step 1: 读取 channel 模块源码结构**

```bash
ls "I:/code/Prism/include/prism/channel/"
```
Expected: transport/, connection/, eyeball/, health.hpp

- [ ] **Step 2: 写入 core/channel/overview.md**

- [ ] **Step 3: 写入 core/channel/health.md**

### Task 18: 创建 core/channel/transport/ 页面

**Files:**
- Create: `H:/PrismWiki/core/channel/transport/transmission.md`
- Create: `H:/PrismWiki/core/channel/transport/reliable.md`
- Create: `H:/PrismWiki/core/channel/transport/encrypted.md`
- Create: `H:/PrismWiki/core/channel/transport/unreliable.md`
- Source: `I:/code/Prism/include/prism/channel/transport/`

- [ ] **Step 1: 读取 transport 源码**

- [ ] **Step 2: 写入各传输层页面**

每个页面包含：
- 源码位置标注
- 功能说明
- 实现细节（复杂函数逐行解释）
- 调用链

### Task 19: 创建 core/channel/connection/, eyeball/, adapter/ 页面

**Files:**
- Create: `H:/PrismWiki/core/channel/connection/pool.md`
- Create: `H:/PrismWiki/core/channel/eyeball/racer.md`
- Create: `H:/PrismWiki/core/channel/adapter/connector.md`
- Source: `I:/code/Prism/include/prism/channel/connection/`, `I:/code/Prism/include/prism/channel/eyeball/`, `I:/code/Prism/include/prism/channel/adapter/`

- [ ] **Step 1: 读取相关源码**

- [ ] **Step 2: 写入 core/channel/connection/pool.md**

- [ ] **Step 3: 写入 core/channel/eyeball/racer.md**

- [ ] **Step 4: 写入 core/channel/adapter/connector.md**

---

## 阶段 8：创建 core/pipeline/ 和 protocol/ 模块页面（并行 3 agent）

### Task 20: 创建 core/pipeline/ 页面

**Files:**
- Create: `H:/PrismWiki/core/pipeline/overview.md`
- Create: `H:/PrismWiki/core/pipeline/primitives.md`
- Create: `H:/PrismWiki/core/pipeline/protocols/http.md`
- Create: `H:/PrismWiki/core/pipeline/protocols/socks5.md`
- Create: `H:/PrismWiki/core/pipeline/protocols/trojan.md`
- Create: `H:/PrismWiki/core/pipeline/protocols/vless.md`
- Create: `H:/PrismWiki/core/pipeline/protocols/shadowsocks.md`
- Source: `I:/code/Prism/include/prism/pipeline/`

### Task 21: 创建 core/protocol/ 页面

**Files:**
- Create: `H:/PrismWiki/core/protocol/overview.md`
- Create: `H:/PrismWiki/core/protocol/common/address.md`
- Create: `H:/PrismWiki/core/protocol/common/form.md`
- Create: `H:/PrismWiki/core/protocol/common/read.md`
- Create: `H:/PrismWiki/core/protocol/http/parser.md`
- Create: `H:/PrismWiki/core/protocol/http/relay.md`
- Create: `H:/PrismWiki/core/protocol/socks5/wire.md`
- Create: `H:/PrismWiki/core/protocol/socks5/stream.md`
- Create: `H:/PrismWiki/core/protocol/socks5/constants.md`
- Create: `H:/PrismWiki/core/protocol/socks5/config.md`
- Create: `H:/PrismWiki/core/protocol/trojan/format.md`
- Create: `H:/PrismWiki/core/protocol/trojan/relay.md`
- Create: `H:/PrismWiki/core/protocol/trojan/constants.md`
- Create: `H:/PrismWiki/core/protocol/trojan/config.md`
- Create: `H:/PrismWiki/core/protocol/vless/format.md`
- Create: `H:/PrismWiki/core/protocol/vless/relay.md`
- Create: `H:/PrismWiki/core/protocol/vless/constants.md`
- Create: `H:/PrismWiki/core/protocol/vless/config.md`
- Create: `H:/PrismWiki/core/protocol/shadowsocks/format.md`
- Create: `H:/PrismWiki/core/protocol/shadowsocks/relay.md`
- Create: `H:/PrismWiki/core/protocol/shadowsocks/salts.md`
- Create: `H:/PrismWiki/core/protocol/shadowsocks/replay.md`
- Create: `H:/PrismWiki/core/protocol/shadowsocks/constants.md`
- Create: `H:/PrismWiki/core/protocol/shadowsocks/config.md`
- Create: `H:/PrismWiki/core/protocol/tls/types.md`
- Create: `H:/PrismWiki/core/protocol/tls/signal.md`
- Create: `H:/PrismWiki/core/protocol/tls/feature_bitmap.md`
- Create: `H:/PrismWiki/core/protocol/analysis.md`
- Source: `I:/code/Prism/include/prism/protocol/`

### Task 22: 创建 core/protocol/ 子目录页面（续）

与 Task 21 合并执行，创建所有 protocol 页面。

---

## 阶段 9：创建 core/stealth/ 模块页面（并行 3 agent）

### Task 23: 创建 core/stealth/ 基础页面

**Files:**
- Create: `H:/PrismWiki/core/stealth/overview.md`
- Create: `H:/PrismWiki/core/stealth/scheme.md`
- Create: `H:/PrismWiki/core/stealth/executor.md`
- Create: `H:/PrismWiki/core/stealth/registry.md`
- Create: `H:/PrismWiki/core/stealth/native.md`
- Source: `I:/code/Prism/include/prism/stealth/`

### Task 24: 创建 core/stealth/reality/ 页面（重点）

**Files:**
- Create: `H:/PrismWiki/core/stealth/reality/scheme.md`
- Create: `H:/PrismWiki/core/stealth/reality/handshake.md`
- Create: `H:/PrismWiki/core/stealth/reality/auth.md`
- Create: `H:/PrismWiki/core/stealth/reality/keygen.md`
- Create: `H:/PrismWiki/core/stealth/reality/request.md`
- Create: `H:/PrismWiki/core/stealth/reality/response.md`
- Create: `H:/PrismWiki/core/stealth/reality/seal.md`
- Create: `H:/PrismWiki/core/stealth/reality/config.md`
- Create: `H:/PrismWiki/core/stealth/reality/constants.md`
- Source: `I:/code/Prism/include/prism/stealth/reality/`, `I:/code/Prism/src/prism/stealth/reality/`

Reality handshake 是复杂模块，需要逐行解释实现逻辑。

### Task 25: 创建 core/stealth/shadowtls/, restls/, anytls/, trusttunnel/, ech/ 页面

**Files:**
- Create: `H:/PrismWiki/core/stealth/shadowtls/scheme.md`
- Create: `H:/PrismWiki/core/stealth/shadowtls/handshake.md`
- Create: `H:/PrismWiki/core/stealth/shadowtls/auth.md`
- Create: `H:/PrismWiki/core/stealth/shadowtls/config.md`
- Create: `H:/PrismWiki/core/stealth/shadowtls/constants.md`
- Create: `H:/PrismWiki/core/stealth/restls/scheme.md`
- Create: `H:/PrismWiki/core/stealth/restls/config.md`
- Create: `H:/PrismWiki/core/stealth/anytls/scheme.md`
- Create: `H:/PrismWiki/core/stealth/anytls/config.md`
- Create: `H:/PrismWiki/core/stealth/trusttunnel/scheme.md`
- Create: `H:/PrismWiki/core/stealth/trusttunnel/config.md`
- Create: `H:/PrismWiki/core/stealth/ech/decrypt.md`
- Create: `H:/PrismWiki/core/stealth/ech/config.md`
- Source: `I:/code/Prism/include/prism/stealth/shadowtls/`, etc.

---

## 阶段 10：创建 core/recognition/ 和 resolve/ 模块页面（并行 3 agent）

### Task 26: 创建 core/recognition/ 页面

**Files:**
- Create: `H:/PrismWiki/core/recognition/overview.md`
- Create: `H:/PrismWiki/core/recognition/recognition.md`
- Create: `H:/PrismWiki/core/recognition/result.md`
- Create: `H:/PrismWiki/core/recognition/confidence.md`
- Create: `H:/PrismWiki/core/recognition/layered_pipeline.md`
- Create: `H:/PrismWiki/core/recognition/probe/probe.md`
- Create: `H:/PrismWiki/core/recognition/probe/analyzer.md`
- Source: `I:/code/Prism/include/prism/recognition/`

### Task 27: 创建 core/resolve/ 页面

**Files:**
- Create: `H:/PrismWiki/core/resolve/overview.md`
- Create: `H:/PrismWiki/core/resolve/router.md`
- Create: `H:/PrismWiki/core/resolve/dns/dns.md`
- Create: `H:/PrismWiki/core/resolve/dns/upstream.md`
- Create: `H:/PrismWiki/core/resolve/dns/config.md`
- Create: `H:/PrismWiki/core/resolve/dns/detail/cache.md`
- Create: `H:/PrismWiki/core/resolve/dns/detail/coalescer.md`
- Create: `H:/PrismWiki/core/resolve/dns/detail/rules.md`
- Create: `H:/PrismWiki/core/resolve/dns/detail/format.md`
- Create: `H:/PrismWiki/core/resolve/dns/detail/utility.md`
- Source: `I:/code/Prism/include/prism/resolve/`, `I:/code/Prism/src/prism/resolve/`

---

## 阶段 11：创建 core/multiplex/, crypto/, outbound/ 模块页面（并行 3 agent）

### Task 28: 创建 core/multiplex/ 页面

**Files:**
- Create: `H:/PrismWiki/core/multiplex/overview.md`
- Create: `H:/PrismWiki/core/multiplex/core.md`
- Create: `H:/PrismWiki/core/multiplex/bootstrap.md`
- Create: `H:/PrismWiki/core/multiplex/duct.md`
- Create: `H:/PrismWiki/core/multiplex/parcel.md`
- Create: `H:/PrismWiki/core/multiplex/config.md`
- Create: `H:/PrismWiki/core/multiplex/smux/craft.md`
- Create: `H:/PrismWiki/core/multiplex/smux/frame.md`
- Create: `H:/PrismWiki/core/multiplex/smux/config.md`
- Create: `H:/PrismWiki/core/multiplex/yamux/craft.md`
- Create: `H:/PrismWiki/core/multiplex/yamux/frame.md`
- Create: `H:/PrismWiki/core/multiplex/yamux/config.md`
- Source: `I:/code/Prism/include/prism/multiplex/`

### Task 29: 创建 core/crypto/ 页面

**Files:**
- Create: `H:/PrismWiki/core/crypto/overview.md`
- Create: `H:/PrismWiki/core/crypto/aead.md`
- Create: `H:/PrismWiki/core/crypto/hkdf.md`
- Create: `H:/PrismWiki/core/crypto/x25519.md`
- Create: `H:/PrismWiki/core/crypto/blake3.md`
- Create: `H:/PrismWiki/core/crypto/block.md`
- Create: `H:/PrismWiki/core/crypto/base64.md`
- Source: `I:/code/Prism/include/prism/crypto/`, `I:/code/Prism/src/prism/crypto/`

### Task 30: 创建 core/outbound/ 页面

**Files:**
- Create: `H:/PrismWiki/core/outbound/overview.md`
- Create: `H:/PrismWiki/core/outbound/proxy.md`
- Create: `H:/PrismWiki/core/outbound/direct.md`
- Source: `I:/code/Prism/include/prism/outbound/`

---

## 阶段 12：创建 core/基础设施模块页面（并行 1 agent）

### Task 31: 创建 core/memory/, fault/, exception/, trace/, transformer/, loader/ 页面

**Files:**
- Create: `H:/PrismWiki/core/memory/overview.md`
- Create: `H:/PrismWiki/core/memory/container.md`
- Create: `H:/PrismWiki/core/memory/pool.md`
- Create: `H:/PrismWiki/core/fault/overview.md`
- Create: `H:/PrismWiki/core/fault/code.md`
- Create: `H:/PrismWiki/core/fault/handling.md`
- Create: `H:/PrismWiki/core/fault/compatible.md`
- Create: `H:/PrismWiki/core/exception/overview.md`
- Create: `H:/PrismWiki/core/exception/deviant.md`
- Create: `H:/PrismWiki/core/exception/network.md`
- Create: `H:/PrismWiki/core/exception/protocol.md`
- Create: `H:/PrismWiki/core/exception/security.md`
- Create: `H:/PrismWiki/core/trace/overview.md`
- Create: `H:/PrismWiki/core/trace/config.md`
- Create: `H:/PrismWiki/core/trace/spdlog.md`
- Create: `H:/PrismWiki/core/transformer/overview.md`
- Create: `H:/PrismWiki/core/transformer/json.md`
- Create: `H:/PrismWiki/core/loader/overview.md`
- Create: `H:/PrismWiki/core/loader/load.md`
- Create: `H:/PrismWiki/core/infrastructure.md`
- Source: `I:/code/Prism/include/prism/memory/`, etc.

---

## 阶段 13：创建 dev/testing/ 和 debugging/ 页面（并行 3 agent）

### Task 32: 创建 dev/testing/ 页面

**Files:**
- Create: `H:/PrismWiki/dev/testing/overview.md`
- Create: `H:/PrismWiki/dev/testing/framework.md`
- Create: `H:/PrismWiki/dev/testing/writing.md`
- Create: `H:/PrismWiki/dev/testing/commands.md`
- Create: `H:/PrismWiki/dev/testing/benchmark.md`
- Create: `H:/PrismWiki/dev/testing/stress.md`
- Create: `H:/PrismWiki/dev/testing/concurrency.md`
- Source: `I:/code/Prism/CLAUDE.md` §测试, `I:/code/Prism/tests/`

### Task 33: 创建 dev/debugging/ 页面

**Files:**
- Create: `H:/PrismWiki/dev/debugging/overview.md`
- Create: `H:/PrismWiki/dev/debugging/connection.md`
- Create: `H:/PrismWiki/dev/debugging/protocol.md`
- Create: `H:/PrismWiki/dev/debugging/memory.md`
- Create: `H:/PrismWiki/dev/debugging/performance.md`
- Create: `H:/PrismWiki/dev/debugging/tls.md`
- Create: `H:/PrismWiki/dev/debugging/log-analysis.md`
- Create: `H:/PrismWiki/dev/debugging/common-issues.md`
- Source: `I:/code/Prism/.claude/skills/proxylog-diagnostic/`, etc.

### Task 34: 创建 dev/performance/, extending/, bugs/ 页面

**Files:**
- Create: `H:/PrismWiki/dev/performance/overview.md`
- Create: `H:/PrismWiki/dev/performance/tuning.md`
- Create: `H:/PrismWiki/dev/performance/profiling.md`
- Create: `H:/PrismWiki/dev/performance/report.md`
- Create: `H:/PrismWiki/dev/extending/overview.md`
- Create: `H:/PrismWiki/dev/extending/protocol.md`
- Create: `H:/PrismWiki/dev/extending/stealth.md`
- Create: `H:/PrismWiki/dev/extending/module.md`
- Create: `H:/PrismWiki/dev/bugs/template.md`
- Create: `H:/PrismWiki/dev/roadmap.md`
- Source: `I:/code/Prism/docs/prism/performance-report.md`, `I:/code/Prism/.claude/skills/protocol-handler/`, `I:/code/Prism/plan.md`

---

## 阶段 14：创建 ref/ 基础页面（并行 3 agent）

### Task 35: 创建 ref/overview 和 ref/crypto/, protocol/, network/ 页面

**Files:**
- Create: `H:/PrismWiki/ref/overview.md`
- Create: `H:/PrismWiki/ref/crypto/overview.md`
- Create: `H:/PrismWiki/ref/crypto/aead-basics.md`
- Create: `H:/PrismWiki/ref/crypto/key-exchange.md`
- Create: `H:/PrismWiki/ref/crypto/hkdf-theory.md`
- Create: `H:/PrismWiki/ref/crypto/tls-crypto.md`
- Create: `H:/PrismWiki/ref/protocol/overview.md`
- Create: `H:/PrismWiki/ref/protocol/tls-handshake.md`
- Create: `H:/PrismWiki/ref/protocol/socks5-spec.md`
- Create: `H:/PrismWiki/ref/protocol/http-proxy-spec.md`
- Create: `H:/PrismWiki/ref/protocol/tcp-basics.md`
- Create: `H:/PrismWiki/ref/protocol/udp-basics.md`
- Create: `H:/PrismWiki/ref/protocol/quic-basics.md`
- Create: `H:/PrismWiki/ref/network/overview.md`
- Create: `H:/PrismWiki/ref/network/happy-eyeballs.md`
- Create: `H:/PrismWiki/ref/network/connection-pool.md`
- Create: `H:/PrismWiki/ref/network/dns-resolution.md`
- Create: `H:/PrismWiki/ref/network/gfw.md`
- Create: `H:/PrismWiki/ref/network/proxy-detection.md`

### Task 36: 创建 ref/mihomo/protocols/ 页面

**Files:**
- Create: `H:/PrismWiki/ref/mihomo/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/socks5.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/socks4.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/http.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/trojan.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/vless.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/vmess.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/shadowsocks.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/shadowsocksr.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/snell.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/ssh.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/hysteria.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/hysteria2.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/tuic.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/wireguard.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/anytls.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/reality.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/ech.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/trusttunnel.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/mieru.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/masque.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/sudoku.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/direct.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/reject.md`
- Create: `H:/PrismWiki/ref/mihomo/protocols/dns.md`
- Source: `I:/code/Prism/logs/mihomo-Meta/adapter/outbound/*.go`

### Task 37: 创建 ref/mihomo/transport/ 和 mux/ 页面

**Files:**
- Create: `H:/PrismWiki/ref/mihomo/transport/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/transport/shadowtls.md`
- Create: `H:/PrismWiki/ref/mihomo/transport/restls.md`
- Create: `H:/PrismWiki/ref/mihomo/transport/v2ray-plugin.md`
- Create: `H:/PrismWiki/ref/mihomo/transport/simple-obfs.md`
- Create: `H:/PrismWiki/ref/mihomo/transport/gost-plugin.md`
- Create: `H:/PrismWiki/ref/mihomo/transport/kcptun.md`
- Create: `H:/PrismWiki/ref/mihomo/transport/gun.md`
- Create: `H:/PrismWiki/ref/mihomo/mux/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/mux/smux.md`
- Create: `H:/PrismWiki/ref/mihomo/mux/yamux.md`
- Create: `H:/PrismWiki/ref/mihomo/mux/singmux.md`
- Create: `H:/PrismWiki/ref/mihomo/mux/config.md`
- Source: `I:/code/Prism/logs/mihomo-Meta/transport/*/`

---

## 阶段 15：创建 ref/mihomo/ 其他模块页面（并行 3 agent）

### Task 38: 创建 ref/mihomo/proxy-groups/, rules/, dns/ 页面

**Files:**
- Create: `H:/PrismWiki/ref/mihomo/proxy-groups/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/proxy-groups/selector.md`
- Create: `H:/PrismWiki/ref/mihomo/proxy-groups/url-test.md`
- Create: `H:/PrismWiki/ref/mihomo/proxy-groups/fallback.md`
- Create: `H:/PrismWiki/ref/mihomo/proxy-groups/load-balance.md`
- Create: `H:/PrismWiki/ref/mihomo/proxy-groups/relay.md`
- Create: `H:/PrismWiki/ref/mihomo/proxy-groups/config.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/domain.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/ipcidr.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/port.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/process.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/geosite.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/geoip.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/rule-set.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/final.md`
- Create: `H:/PrismWiki/ref/mihomo/rules/logic.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/servers.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/enable.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/listen.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/enhanced-mode.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/fake-ip-filter.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/nameserver-policy.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/fallback.md`
- Create: `H:/PrismWiki/ref/mihomo/dns/fallback-filter.md`
- Source: `I:/code/Prism/logs/mihomo-Meta/adapter/outboundgroup/*.go`, `I:/code/Prism/logs/mihomo-Meta/rules/*.go`, `I:/code/Prism/logs/mihomo-Meta/dns/*.go`

### Task 39: 创建 ref/mihomo/tun/, config/, compatibility/, implementation/ 页面

**Files:**
- Create: `H:/PrismWiki/ref/mihomo/tun/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/tun/enable.md`
- Create: `H:/PrismWiki/ref/mihomo/tun/stack.md`
- Create: `H:/PrismWiki/ref/mihomo/tun/dns-hijack.md`
- Create: `H:/PrismWiki/ref/mihomo/tun/auto-route.md`
- Create: `H:/PrismWiki/ref/mihomo/tun/auto-detect-interface.md`
- Create: `H:/PrismWiki/ref/mihomo/config/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/config/basic.yaml`
- Create: `H:/PrismWiki/ref/mihomo/config/advanced.yaml`
- Create: `H:/PrismWiki/ref/mihomo/config/mux.yaml`
- Create: `H:/PrismWiki/ref/mihomo/config/tun.yaml`
- Create: `H:/PrismWiki/ref/mihomo/config/dns.yaml`
- Create: `H:/PrismWiki/ref/mihomo/config/rules.yaml`
- Create: `H:/PrismWiki/ref/mihomo/config/proxy-groups.yaml`
- Create: `H:/PrismWiki/ref/mihomo/config/full-example.yaml`
- Create: `H:/PrismWiki/ref/mihomo/compatibility/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/compatibility/prism.md`
- Create: `H:/PrismWiki/ref/mihomo/compatibility/features.md`
- Create: `H:/PrismWiki/ref/mihomo/compatibility/protocol-matrix.md`
- Create: `H:/PrismWiki/ref/mihomo/compatibility/transport-matrix.md`
- Create: `H:/PrismWiki/ref/mihomo/implementation/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/implementation/udp-relay.md`
- Create: `H:/PrismWiki/ref/mihomo/implementation/tcp-concurrent.md`
- Create: `H:/PrismWiki/ref/mihomo/implementation/keepalive.md`
- Create: `H:/PrismWiki/ref/mihomo/implementation/authentication.md`

### Task 40: 创建 ref/mihomo/sniffing/, listeners/, provider/, ntp/, experimental/, script/ 页面

**Files:**
- Create: `H:/PrismWiki/ref/mihomo/sniffing/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/sniffing/enable.md`
- Create: `H:/PrismWiki/ref/mihomo/sniffing/sniffing-types.md`
- Create: `H:/PrismWiki/ref/mihomo/sniffing/ports.md`
- Create: `H:/PrismWiki/ref/mihomo/listeners/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/listeners/mixed.md`
- Create: `H:/PrismWiki/ref/mihomo/listeners/socks.md`
- Create: `H:/PrismWiki/ref/mihomo/listeners/http.md`
- Create: `H:/PrismWiki/ref/mihomo/listeners/tun.md`
- Create: `H:/PrismWiki/ref/mihomo/listeners/redir.md`
- Create: `H:/PrismWiki/ref/mihomo/listeners/tproxy.md`
- Create: `H:/PrismWiki/ref/mihomo/provider/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/provider/proxy-provider.md`
- Create: `H:/PrismWiki/ref/mihomo/provider/rule-provider.md`
- Create: `H:/PrismWiki/ref/mihomo/provider/health-check.md`
- Create: `H:/PrismWiki/ref/mihomo/provider/override.md`
- Create: `H:/PrismWiki/ref/mihomo/ntp/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/ntp/enable.md`
- Create: `H:/PrismWiki/ref/mihomo/ntp/server.md`
- Create: `H:/PrismWiki/ref/mihomo/experimental/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/experimental/quic-go.md`
- Create: `H:/PrismWiki/ref/mihomo/experimental/udp-over-tcp.md`
- Create: `H:/PrismWiki/ref/mihomo/script/overview.md`
- Create: `H:/PrismWiki/ref/mihomo/script/shortcuts.md`
- Create: `H:/PrismWiki/ref/mihomo/script/rule-script.md`
- Create: `H:/PrismWiki/ref/mihomo/index.md`
- Source: `I:/code/Prism/logs/mihomo-Meta/` 各目录

---

## 阶段 16：创建 ref/programming/ 和 glossary 页面（并行 1 agent）

### Task 41: 创建 ref/programming/ 和 glossary 页面

**Files:**
- Create: `H:/PrismWiki/ref/programming/overview.md`
- Create: `H:/PrismWiki/ref/programming/boost-asio.md`
- Create: `H:/PrismWiki/ref/programming/cpp23-coroutine.md`
- Create: `H:/PrismWiki/ref/programming/constexpr.md`
- Create: `H:/PrismWiki/ref/programming/pmr-concepts.md`
- Create: `H:/PrismWiki/ref/programming/error-handling.md`
- Create: `H:/PrismWiki/ref/programming/go-concurrency.md`
- Create: `H:/PrismWiki/ref/glossary.md`
- Source: `I:/code/Prism/CLAUDE.md`, 知识积累

---

## 阶段 17：创建 wiki-sync skill（并行 1 agent）

### Task 42: 创建 skills/prism-wiki-sync/ 所有文件

**Files:**
- Create: `H:/PrismWiki/skills/prism-wiki-sync/SKILL.md`
- Create: `H:/PrismWiki/skills/prism-wiki-sync/checks.md`
- Create: `H:/PrismWiki/skills/prism-wiki-sync/report-template.md`
- Create: `H:/PrismWiki/skills/prism-wiki-sync/references/prism-structure.md`
- Create: `H:/PrismWiki/skills/prism-wiki-sync/references/mihomo-structure.md`
- Create: `H:/PrismWiki/skills/prism-wiki-sync/references/wiki-structure.md`

- [ ] **Step 1: 写入 SKILL.md**

```markdown
---
name: prism-wiki-sync
description: 检查知识库与源码同步状态。Push时全面检查，Release时深度检查。
---

# prism-wiki-sync Skill

## 触发条件

- Push 到远程仓库
- Release (tag v*)
- 手动 `/wiki-sync`

## 检查模式

- `normal`: Push 时的全面检查（11项）
- `deep`: Release 时的深度检查（+5项）

## 输出格式

- 严重问题 → 终端输出（直接打印）
- 完整报告 → `.claude/wiki-sync-report.md`

## 源码位置

- Prism: `I:/code/Prism/`
- Mihomo: `I:/code/Prism/logs/mihomo-Meta/`

## 检查项概览

### 普通检查（Push）

1. 源码路径验证
2. 函数签名变化
3. 调用链变化
4. 枚举/常量变化
5. 类结构变化
6. 配置项变化
7. 错误码变化
8. 协议兼容性
9. 测试覆盖
10. 依赖关系变化
11. 文档结构

### 深度检查（Release）

1. 文档一致性
2. 链接完整性
3. 覆盖度检查
4. 变更影响分析
5. 新功能检测

详细检查项见 [[checks]]
```

- [ ] **Step 2: 写入 checks.md**

详细内容见下方。

- [ ] **Step 3: 写入 report-template.md**

- [ ] **Step 4: 写入 references/prism-structure.md**

- [ ] **Step 5: 写入 references/mihomo-structure.md**

- [ ] **Step 6: 写入 references/wiki-structure.md**

---

## 阶段 18：清理旧目录/文件（并行 1 agent）

### Task 43: 删除旧目录结构

**Files:**
- Delete: `H:/PrismWiki/incoming/`
- Delete: `H:/PrismWiki/recognition/recognition/`
- Delete: `H:/PrismWiki/agent/`（旧结构）
- Delete: `H:/PrismWiki/channel/`（旧结构）
- Delete: `H:/PrismWiki/crypto/`（旧结构）
- Delete: `H:/PrismWiki/multiplex/`（旧结构）
- Delete: `H:/PrismWiki/pipeline/`（旧结构）
- Delete: `H:/PrismWiki/protocol/`（旧结构）
- Delete: `H:/PrismWiki/stealth/`（旧结构）
- Delete: `H:/PrismWiki/resolve/`（旧结构）
- Delete: `H:/PrismWiki/memory/`（旧结构）
- Delete: `H:/PrismWiki/fault/`（旧结构）
- Delete: `H:/PrismWiki/exception/`（旧结构）
- Delete: `H:/PrismWiki/trace/`（旧结构）
- Delete: `H:/PrismWiki/transformer/`（旧结构）
- Delete: `H:/PrismWiki/loader/`（旧结构）
- Delete: `H:/PrismWiki/outbound/`（旧结构）
- Delete: `H:/PrismWiki/client/`（旧结构）
- Delete: `H:/PrismWiki/dev/`（旧结构）
- Delete: `H:/PrismWiki/performance/`（旧结构）
- Delete: `H:/PrismWiki/.plan/`
- Delete: `H:/PrismWiki/docs/agent/`
- Delete: `H:/PrismWiki/docs/channel/`
- Delete: `H:/PrismWiki/docs/crypto/`
- Delete: `H:/PrismWiki/docs/multiplex/`
- Delete: `H:/PrismWiki/docs/pipeline/`
- Delete: `H:/PrismWiki/docs/protocol/`
- Delete: `H:/PrismWiki/docs/stealth/`
- Delete: `H:/PrismWiki/docs/resolve/`

- [ ] **Step 1: 列出要删除的目录**

```bash
ls -la "H:/PrismWiki/" | grep -E "^d"
```
Expected: 当前目录结构

- [ ] **Step 2: 删除旧顶层目录**

```bash
rm -rf "H:/PrismWiki/incoming"
rm -rf "H:/PrismWiki/agent"
rm -rf "H:/PrismWiki/channel"
rm -rf "H:/PrismWiki/crypto"
rm -rf "H:/PrismWiki/multiplex"
rm -rf "H:/PrismWiki/pipeline"
rm -rf "H:/PrismWiki/protocol"
rm -rf "H:/PrismWiki/stealth"
rm -rf "H:/PrismWiki/resolve"
rm -rf "H:/PrismWiki/memory"
rm -rf "H:/PrismWiki/fault"
rm -rf "H:/PrismWiki/exception"
rm -rf "H:/PrismWiki/trace"
rm -rf "H:/PrismWiki/transformer"
rm -rf "H:/PrismWiki/loader"
rm -rf "H:/PrismWiki/outbound"
rm -rf "H:/PrismWiki/client"
rm -rf "H:/PrismWiki/performance"
rm -rf "H:/PrismWiki/.plan"
```

- [ ] **Step 3: 删除 docs/ 旧子目录**

```bash
rm -rf "H:/PrismWiki/docs/agent"
rm -rf "H:/PrismWiki/docs/channel"
rm -rf "H:/PrismWiki/docs/crypto"
rm -rf "H:/PrismWiki/docs/multiplex"
rm -rf "H:/PrismWiki/docs/pipeline"
rm -rf "H:/PrismWiki/docs/protocol"
rm -rf "H:/PrismWiki/docs/stealth"
rm -rf "H:/PrismWiki/docs/resolve"
```

- [ ] **Step 4: 删除重复的 recognition/recognition/ 目录**

```bash
rm -rf "H:/PrismWiki/recognition/recognition"
```

- [ ] **Step 5: 验证清理结果**

```bash
ls -la "H:/PrismWiki/"
```
Expected: 只剩 core/, dev/, docs/, ref/, skills/, SCHEMA.md, index.md, log.md, README.md

---

## 阶段 19：更新 log.md 并提交（并行 1 agent）

### Task 44: 更新 log.md 并提交

**Files:**
- Modify: `H:/PrismWiki/log.md`

- [ ] **Step 1: 读取现有 log.md**

```bash
head -50 "H:/PrismWiki/log.md"
```

- [ ] **Step 2: 写入新的操作记录**

记录本次重构的所有操作。

- [ ] **Step 3: 提交所有更改**

```bash
cd "H:/PrismWiki"
git add .
git commit -m "refactor: 知识库四层重构 - core/dev/docs/ref + wiki-sync skill

- 重构目录结构：core/(详细), dev/(开发), docs/(用户), ref/(参考+mihomo)
- 重写 SCHEMA.md：定义各层详细程度、wikilink规范、源码标注规范
- 重写 index.md：更新为新目录结构索引
- 创建 core/ 所有模块页面：agent/channel/pipeline/protocol/stealth/recognition/resolve/multiplex/crypto/outbound/memory/fault/exception/trace/transformer/loader
- 创建 dev/ 所有页面：coding/testing/debugging/building/performance/extending/bugs/roadmap
- 创建 docs/ 用户指南页面：getting-started/deployment/configuration/client-setup/faq/troubleshooting/upgrade/security/performance-tips
- 创建 ref/ 参考知识页面：crypto/protocol/network/programming/glossary
- 创建 ref/mihomo/ 完整参考：protocols(22个)/transport(7个)/mux/proxy-groups/rules/dns/tun/config/compatibility/implementation/sniffing/listeners/provider/ntp/experimental/script
- 创建 skills/prism-wiki-sync/：SKILL.md/checks.md/report-template.md/references/
- 删除旧目录结构：incoming/agent/channel/crypto/multiplex/pipeline/protocol/stealth/resolve/memory/fault/exception/trace/transformer/loader/outbound/client/dev/performance/.plan/docs旧子目录

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>"
```

---

## 验收标准

完成后应满足：

1. [ ] 目录结构完整：core/dev/docs/ref/skills 存在且子目录完整
2. [ ] SCHEMA.md 包含完整规范定义
3. [ ] index.md 是完整索引，链接到各层 overview
4. [ ] core/ 所有模块页面存在，内容基于源码
5. [ ] dev/ 所有页面存在，内容基于 CLAUDE.md
6. [ ] docs/ 所有页面存在，内容简化
7. [ ] ref/ 所有页面存在，ref/mihomo/ 包含完整协议参考
8. [ ] skills/prism-wiki-sync/ 所有文件存在
9. [ ] 旧目录已清理
10. [ ] 无重复知识（同一知识点只一处详细写）
11. [ ] wikilink 正确指向目标页面

---

## 执行顺序图

```
阶段1 [1,2,3] → 目录创建
阶段2 [4,5,6] → SCHEMA/index/core-overview
阶段3 [7,8,9] → architecture/startup/flow
阶段4 [10,11,12] → dev基础
阶段5 [13] → docs
阶段6 [14,15,16] → agent
阶段7 [17,18,19] → channel
阶段8 [20,21,22] → pipeline+protocol
阶段9 [23,24,25] → stealth
阶段10 [26,27] → recognition+resolve
阶段11 [28,29,30] → multiplex+crypto+outbound
阶段12 [31] → 基础设施
阶段13 [32,33,34] → testing+debugging+其他
阶段14 [35,36,37] → ref基础+mihomo协议
阶段15 [38,39,40] → mihomo其他
阶段16 [41] → programming+glossary
阶段17 [42] → skill
阶段18 [43] → 清理
阶段19 [44] → 提交
```

---

**计划完成。请审核后批准执行。**