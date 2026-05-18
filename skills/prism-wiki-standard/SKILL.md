---
name: prism-wiki-standard
description: "Use when creating, editing, auditing, or batch-refreshing PrismWiki documentation. Covers document type system, function-level API docs, C++ specifics, file/folder operations, link management, and quality tiers."
version: 2.0.0
author: Wang
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [wiki, prism, knowledge-base, obsidian, documentation, cpp]
    related_skills: [wiki-knowledge-base, obsidian, project-knowledge-base]
layer: skills
---

# PrismWiki 知识库规范

PrismWiki（`H:\PrismWiki`）是 Prism 高性能协程代理引擎（C++23 + Boost.Asio + PMR）的 Obsidian 知识库。本 skill 定义文档标准、文件操作规范、质量分档、审计流程。

- **源码**: `I:\code\Prism\` — 头文件 `include/prism/<module>/`，实现 `src/prism/<module>/`
- **Wiki**: `H:\PrismWiki\` — 当前 ~221 个 md 文件，~60 个目录
- **语言**: 全中文，代码标识符保持英文并用反引号包裹

## 文档类型体系

Wiki 中有 7 种文档类型，各有不同的 frontmatter、段落要求和适用范围：

| 类型 | type 值 | 适用目录 | 说明 |
|------|---------|----------|------|
| API 文档 | `api` | agent/, channel/, crypto/, multiplex/, pipeline/, protocol/, recognition/, resolve/, stealth/ + 根目录 | 对应 hpp 文件，函数级深度 |
| 模块概览 | `module` | 根目录小型模块 | loader.md, transformer.md 等多 hpp 合并文件 |
| 协议文档 | `protocol` | docs/stealth/, docs/protocol/ | 概念性协议分析，不对应具体 hpp |
| 知识页 | `ref` | ref/ | 外部知识：密码学、协议、网络、编程 |
| 开发指南 | `dev` | dev/ | 构建、测试、调试、C++ 笔记 |
| 客户端配置 | `client` | client/ | mihomo/tun 对接、配置模板 |
| 性能报告 | `performance` | performance/ | 基准测试、性能分析报告 |

**判断方法**：看文件是否对应一个具体 hpp。是 → `api` 或 `module`；不是 → 按内容选类型。

---

## 类型 1：API 文档（核心）

每个 hpp 文件对应一个 wiki md 文件，函数在文件内用 `## 函数:` 分节。这是最主要的文档类型。

### 文件命名与路径

```
源码: include/prism/stealth/shadowtls/auth.hpp
Wiki: stealth/shadowtls/auth.md

规则: 去掉 include/prism/ 前缀，.hpp → .md
```

### Frontmatter

```yaml
---
title: "auth — ShadowTLS v3 认证逻辑"        # 文件名 — 一句话中文描述
source: "include/prism/stealth/shadowtls/auth.hpp"  # 相对路径，不标行号
module: "stealth"                             # 第一层目录名，不是 C++ 命名空间
type: api
tags: [stealth, shadowtls, auth, hmac, 认证]  # 3-6 个，模块+功能+关键词
created: 2026-05-15
updated: 2026-05-15                           # 每次修改必须更新
related:
  - "[[core/stealth/shadowtls/handshake]]"         # 2-5 个最相关的页面
  - "[[ref/crypto/hmac-sha1]]"
---
```

**注意**：
- `module` 必须匹配第一层目录名（`stealth` 不是 `stealth::shadowtls`）
- `source` 用相对路径 `include/prism/...`（相对于 Prism 项目根 `I:\code\Prism\`），不标行号
- 如果有实现文件，在概述段落里写，不在 frontmatter 加字段

### 必有段落（按顺序）

#### 1. 标题与源码引用

```markdown
# auth.hpp

> 源码: `include/prism/stealth/shadowtls/auth.hpp`
> 实现: `src/prism/stealth/shadowtls/auth.cpp`        # 有才写
> 模块: [[core/stealth|stealth]] > [[core/stealth/shadowtls|shadowtls]]
```

#### 2. 概述

一段话说明文件职责、设计意图、在系统中的角色。**必须读 hpp 的 `@brief` 再写**，不要靠猜。

#### 3. 命名空间

```markdown
## 命名空间

`psm::stealth::shadowtls`
```

#### 4. 依赖关系

```markdown
## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[core/crypto/hkdf]] | 密钥派生 |
| 继承 | [[core/channel/transport/transmission]] | 传输层接口 |
| 被依赖 | [[core/agent/session/session|session]] | session 启动时调用 |
```

依赖方向填写：`依赖` / `继承` / `组合` / `被依赖` / `被调用`

#### 5. 类/结构体详解

**类与结构体**：API 文档的核心类必须有以下格式：

```markdown
### 类: relay

Trojan 协议中继器。装饰器模式，包装底层传输层。

**成员变量**:
| 类型 | 名称 | 说明 |
|------|------|------|
| shared_transmission | inner_ | 被包装的传输层 |
| config const& | config_ | 协议配置引用 |

**关键方法**:
- [[#函数: async_read_some]] — 异步读取
- [[#函数: async_write_some]] — 异步写入
```

**子类/继承关系**：如果类有基类，在类说明第一行标注 `继承自 [[base_class]]`。

#### 6. 配置项说明

如果文件有 config 结构体，必须有配置项表：

```markdown
## 配置项

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| password | string | — | 认证密码 |
| strict_mode | bool | false | 强制 TLS 1.3 |
```

没有配置结构体的文件可以省略此段落。

**源码路径**：API 文档每个函数可以在函数分节头部标注源码位置，格式为 `> 源码: \`include/prism/.../file.hpp:行号\``。行号可选，不强制要求。

**函数签名表**：API 文档必须有函数签名表，放在「配置项」（或「类与结构体」）之后。格式：

```markdown
## 函数签名表

| 函数 | 签名 | 说明 |
|------|------|------|
| [[#函数: verify_client_hello]] | `auto verify_client_hello(...) -> bool` | 验证 ClientHello |
| [[#函数: compute_hmac]] | `auto compute_hmac(...) -> void` | 计算 HMAC 标签 |
```

链接目标用 `[[#函数: 函数名]]` 格式。如果文件无公开函数，写「无公开函数」。

函数分节格式统一为以下两种（根据实际项目惯例）：

**模式 A（API 文档 — 函数级）**：
```markdown
## 函数: verify_client_hello()

> 源码: `include/prism/stealth/shadowtls/auth.hpp:36`

**功能**: 验证 TLS ClientHello 中 SessionID 的 HMAC-SHA1 标签

**签名**:
`auto verify_client_hello(buffer_view hello, span<const uint8_t> password) -> bool`
```

**模式 B（模块概览 — 多 hpp 合并文件内）**：
```markdown
### 函数: load()

- **功能**: 从文件系统加载配置文件
- **签名**: `inline auto load(std::string_view path) -> config`
```

**选择规则**：
- API 文档（type: api）用**模式 A**（`##` + 函数签名头 + 详情块）
- 模块概览（type: module）内各 hpp 的函数用**模式 B**（`###` + 简洁列表格式）
- 构造函数和析构函数：简单构造函数可合并到类的概述中，不必单独分节

**函数详解规则**：
- 「功能」「签名」「参数」「返回值」「调用链」必有
- 「知识域」可选
- 调用链必须 grep 整个项目验证，不要凭记忆
- 构造函数、析构函数不需要单独分节，除非有复杂逻辑
- 枚举值、类型别名可以用表格代替分节

**调用链格式**（两种均可，根据文件已有格式选择）：

格式 A（函数内联段落，推荐新建文件使用）：
```markdown
**调用链**:
- **调用**（向下）:
  - [[#函数: compute_hmac]] — 计算期望的 HMAC 值
- **被调用**（向上）:
  - [[core/stealth/shadowtls/handshake#函数: on_client_hello]] — 握手阶段调用
```

格式 B（独立 `###` 小标题，项目中已有文件更常见）：
```markdown
### 调用（向下）
- [[#函数: compute_hmac]] — 计算期望的 HMAC 值

### 被调用（向上）
- [[core/stealth/shadowtls/handshake#函数: on_client_hello]] — 握手阶段调用
```

---

### C++ 特定文档规则

#### 模板与 CRTP

```markdown
### 类模板: `pooled_object`

CRTP 对象池基类。子类自动获得内存池分配能力。

**签名**: `template <typename T, pool_type Type = pool_type::local> class pooled_object`

**模板参数**:
| 参数 | 约束 | 说明 |
|------|------|------|
| T | 继承 pooled_object 的类 | CRTP 子类类型 |
| Type | pool_type 枚举 | 池类型选择 |
```

#### 协程函数

Boost.Asio 协程函数标注返回类型为 `awaitable<>`：

```markdown
**签名**: `auto async_connect(target const& t, executor const& ex) -> awaitable<pair<fault::code, shared_transmission>>`

**协程说明**: 无栈协程，co_await 内部 IO 操作
```

#### 虚函数

```markdown
### 函数: async_connect [pure virtual]

### 函数: async_connect [override]
```

方括号标注 `[pure virtual]` / `[virtual]` / `[override]` / `[static]` / `[private]`

#### 枚举

```markdown
### 枚举: `pool_type`

内存池类型选择。

| 值 | 说明 |
|----|------|
| `global` | 全局线程安全池 |
| `local` | 线程局部无锁池 |
```

#### 类型别名

```markdown
### 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `resource` | `std::pmr::memory_resource` | 内存资源基类 |
| `vector<Value>` | `std::pmr::vector<Value>` | PMR 动态数组 |
```

#### 操作符重载

```markdown
### 函数: operator new()

**功能**: 重载单对象 new 操作符。小对象从目标池分配，大对象直通系统堆。
**签名**: `void *operator new(std::size_t count)`
```

---

## 类型 2：模块概览（多 hpp 合并）

适用于小型基础设施模块（loader, transformer, memory, outbound 等），多个 hpp 合并到一个 md 文件中。

**结构**：每个 hpp 用 `## hpp 文件名` 分节，节内包含该 hpp 的概述、命名空间、类、函数。

**示例**（loader.md）：
```markdown
## load.hpp — 配置加载适配器

> 源码: `include/prism/loader/load.hpp`

### 概述
配置加载适配器...

### 命名空间
`psm::loader`

### 函数: load()
...

---

## build.hpp — 账户目录构建

> 源码: `include/prism/loader/build.hpp`
...
```

**判断是否合并**：
- 模块 hpp ≤ 3 个且每个 < 150 行 → 合并到一个 md
- 模块 hpp > 3 个或有复杂子模块 → 每个 hpp 单独 md

---

## 类型 3：协议文档（docs/）

概念性协议分析，不对应具体 hpp。放在 `docs/` 目录下。

```yaml
---
title: ShadowTLS
type: protocol
tags: [stealth, tls, hmac]
---
```

**必有段落**：概述、工作原理、版本差异（如有）、与 Prism 实现的关系、相关页面。

**不需要**：函数签名表、调用链、参数表。

---

## 类型 4：知识页（ref/）

外部知识点参考。放在 `ref/` 目录下，按类别分目录：`crypto/` `protocol/` `network/` `memory/` `programming/` `anti-censorship/`。

```yaml
---
title: "HMAC-SHA1"
category: "crypto"
type: ref
tags: [密码学, hmac, sha1]
---
```

**必有段落**：
1. 概述 — 一句话说明
2. 原理 — 详细技术说明
3. 在 Prism 中的应用 — 表格列出哪些函数使用了此知识，链接到 API 文档
4. 参考资料 — RFC、论文链接
5. 相关知识 — wikilink 到其他 ref 页

---

## 类型 5：开发指南（dev/）

构建、测试、调试、C++ 编程笔记。

```yaml
---
title: 测试体系
tags: [testing, ctest, unit, e2e]
---
```

**没有** frontmatter 的 `module` 和 `source` 字段。自由格式，但必须有 `tags` 和至少 2 个出站 wikilink。

---

## 类型 6：客户端配置（client/）

mihomo 对接、TUN 配置、Clash 模板。

```yaml
---
title: mihomo 配置模板
type: client
tags: [client, mihomo, clash, config]
---
```

---

## 类型 7：性能报告（performance/）

基准测试结果、性能分析报告。

```yaml
---
title: 基准测试报告
type: performance
tags: [performance, benchmark, latency, throughput]
---
```

**必有段落**：测试环境、测试方法、结果数据、分析结论。

---

## 详细度分档

不是所有文件都需要同等详细。按 hpp 复杂度分三档：

### S 档（核心模块，300+ 行）

**适用**：stealth（ShadowTLS/Reality/Restls/ECH）、crypto/aead、protocol 各 relay、agent/session、channel/transport/snapshot、multiplex/core

**要求**：
- 所有必有段落全部填写，一个不少
- 每个函数完整展开：参数表、返回值、调用链（上下都写）
- 类/结构体列出所有成员变量（包括 private，如果对理解设计有帮助）
- 知识域尽量补充
- 配置项逐字段说明

### A 档（标准模块，150-300 行）

**适用**：protocol 各 config/constants/format、multiplex/smux|yamux、resolve/dns 子模块、agent 子模块

**要求**：
- 所有必有段落填写
- 函数有签名和简要说明
- 调用链只列直接上下游（不用递归追溯）
- 类/结构体列出关键成员
- 配置项列表说明

### B 档（简单模块，50-150 行）

**适用**：纯 config 文件、constants 文件、简单工具类（base64, sha224, block）、format/parse 辅助

**要求**：
- 概述、命名空间、依赖关系必有
- 类/结构体可以只列名称和一句话说明
- 函数签名表必有，函数详解可以几个小函数合并写
- 调用链可以省略（函数太简单时）

### 分档判断

```bash
# hpp 行数 + 函数数量
wc -l include/prism/<module>/<file>.hpp
grep -cE 'auto |void |static |virtual |class |struct ' include/prism/<module>/<file>.hpp
```

- hpp > 200 行 且 关键符号 > 15 → S 档
- hpp 80-200 行 或 关键符号 8-15 → A 档
- hpp < 80 行 或 关键符号 < 8 → B 档

---

## 文件操作规范

### 创建新文件

**场景**：Prism 源码新增了 hpp，需要创建对应 wiki 页面。

**步骤**：
1. 读 hpp 源码（必须读，不要靠猜）
2. 读对应的 cpp（如果有）
3. 确定文档类型（api / module / ref / ...）
4. 确定详细度档位（S / A / B）
5. 按模板填写 frontmatter + 所有必有段落
6. grep 项目补充调用链
7. 更新 `index.md` 添加新页面链接
8. 更新 `log.md` 记录操作

**检查清单**：
- [ ] frontmatter 字段完整（title, source, module, type, tags, created, updated, related）
- [ ] module 字段匹配第一层目录名
- [ ] 概述与 hpp @brief 一致
- [ ] 所有公开函数都有文档
- [ ] 调用链经过 grep 验证
- [ ] 已添加到 index.md
- [ ] 至少 3 个出站 wikilink

### 删除文件

**场景**：源码中删除了 hpp，或合并/重构导致页面不再需要。

**步骤**：
1. 用 `grep -rn "filename" H:/PrismWiki/ --include="*.md"` 找所有引用此页面的地方
2. 清理所有引用：删除 wikilink、更新相关段落
3. 从 `index.md` 移除
4. 删除文件
5. 更新 `log.md`
6. 检查是否有目录变成空目录（空目录要删除）

**检查清单**：
- [ ] 所有引用已清理（grep 确认无残留）
- [ ] index.md 已更新
- [ ] 无空目录残留
- [ ] log.md 已记录

### 移动/重命名文件

**场景**：模块重构、目录结构调整。

**步骤**：
1. 用 grep 找所有引用旧路径的地方
2. 创建新文件（复制内容，更新 frontmatter）
3. 批量更新所有引用：`patch` 工具 + `replace_all=true`
4. 从 index.md 移除旧条目，添加新条目
5. 删除旧文件
6. 检查空目录
7. 更新 log.md

**批量替换旧路径**：
```python
# 用 execute_code 批量替换
from hermes_tools import search_files, patch

# 找所有包含旧路径的文件
results = search_files("old/path/name", target="content", path="H:/PrismWiki")
for match in results.get("matches", []):
    f = match["file"]
    patch(f, "[[old/path/name", "[[new/path/name", replace_all=True)
```

### 创建新目录

**场景**：新增模块需要新目录。

**规则**：
- 目录名小写，连字符分隔（如 `stealth/anytls/`）
- 与源码 `include/prism/` 下的目录结构对应
- 不需要 `_index.md`（用户明确不要）
- 新目录下至少要有一个 md 文件，不允许空目录

**步骤**：
1. `mkdir -p H:/PrismWiki/<path>`
2. 创建第一个 md 文件
3. 更新 index.md 添加目录下的页面
4. 更新 log.md

### 删除目录

**规则**：只有目录下所有文件都已删除时才能删目录。

**步骤**：
1. 确认目录下无 md 文件：`ls H:/PrismWiki/<path>/`
2. `rmdir H:/PrismWiki/<path>/`
3. 更新 log.md

---

## 链接管理

### Wikilink 格式

```markdown
[[target]]                    # 链接到 target.md（同级或根目录）
[[module/subtarget]]          # 链接到 module/subtarget.md
[[module/file#函数名]]        # 链接到特定函数分节
[[target|显示名]]             # 自定义显示文本
```

**注意**：C++ 属性如 `[[likely]]`、`[[nodiscard]]`、`[[maybe_unused]]` 不是 wikilink，grep 时要过滤。

### 链接密度

- 每个页面至少 3 个出站 wikilink（不含 frontmatter related）
- related 字段建议 2-5 个
- 不能有死链（目标页面必须存在）
- 不能有孤儿页（每个页面至少被 index.md 链接一次）

### 新建页面后

1. 检查哪些页面应该链接到新页面（被依赖的模块）
2. 用 `patch` 在那些页面的「依赖关系」或「相关页面」中添加反向链接
3. 在 index.md 中添加链接

### 批量链接检查

```bash
# 提取所有 wikilink 目标（过滤 C++ 属性）
grep -roh '\[\[[^]]*\]\]' H:/PrismWiki/ --include="*.md" | \
  sed 's/\[\[//;s/\]\]//' | sed 's/|.*//' | sed 's/#.*//' | \
  sort | uniq -c | sort -rn

# 找死链（目标文件不存在）
grep -roh '\[\[[^]]*\]\]' H:/PrismWiki/ --include="*.md" | \
  sed 's/\[\[//;s/\]\]//' | sed 's/|.*//' | sed 's/#.*//' | \
  sort -u | while read target; do
    [ ! -f "H:/PrismWiki/${target}.md" ] && echo "DEAD: $target"
  done
```

---

## Index.md 管理

index.md 是全库入口页，按模块分组列出所有内容页面。

### 结构

```markdown
## 核心模块

- [[memory]] — PMR 内存池
- [[fault]] — 错误码枚举
...

## Agent 模块

- [[core/agent/config|config]] — Agent 运行时配置
- [[core/agent/session/session|session]] — 会话管理
...
```

### 更新规则

- 新建页面 → 必须添加到 index.md 对应分组
- 删除页面 → 必须从 index.md 移除
- 移动页面 → 更新 index.md 中的链接路径
- index.md 中的分组标题要与目录结构对应

---

## 审计与验证

### 快速健康检查

```bash
# 总文件数
find H:/PrismWiki -name '*.md' -not -path '*/.git/*' -not -path '*/.obsidian/*' -not -path '*/.plan/*' -not -path '*/skills/*' | wc -l

# 按目录统计
find H:/PrismWiki -name '*.md' -not -path '*/.git/*' -not -path '*/.obsidian/*' -not -path '*/.plan/*' -not -path '*/skills/*' -exec dirname {} \; | sort | uniq -c | sort -rn

# 质量分布
find H:/PrismWiki -name '*.md' -not -path '*/.git/*' -not -path '*/.obsidian/*' -not -path '*/.plan/*' -not -path '*/skills/*' -print0 | xargs -0 wc -l | grep -v total | awk '{if($1<50)thin++;else if($1<150)med++;else rich++}END{print "B档(<50):",thin;print "A档(50-150):",med;print "S档(>150):",rich}'
```

### 段落完整性检查

```bash
# 检查 API 文档缺少「函数签名表」的文件
for f in $(find H:/PrismWiki -name '*.md' -not -path '*/.git/*' -not -path '*/.obsidian/*' -not -path '*/.plan/*' -not -path '*/skills/*' -not -path '*/ref/*' -not -path '*/dev/*' -not -path '*/client/*' -not -path '*/docs/*' -not -path '*/performance/*'); do
  type=$(head -15 "$f" | grep -m1 '^type:' | sed 's/^type: *//')
  [ "$type" = "api" ] && ! grep -q '函数签名表\|函数:' "$f" && echo "MISSING_SIG: $f"
done

# 检查缺少「调用链」的 API 文件（S/A 档应该有）
for f in $(find H:/PrismWiki -name '*.md' -not -path '*/.git/*' -not -path '*/.obsidian/*' -not -path '*/.plan/*' -not -path '*/skills/*' -not -path '*/ref/*' -not -path '*/dev/*' -not -path '*/client/*' -not -path '*/docs/*' -not -path '*/performance/*'); do
  lines=$(wc -l < "$f")
  [ "$lines" -gt 150 ] && ! grep -q '调用链\|调用（向下）' "$f" && echo "MISSING_CHAIN($lines): $f"
done
```

### 源码覆盖率审计

详见 `references/source-coverage-audit.md`，对比 wiki 页面与 hpp 文件的覆盖情况。

### 描述准确性审计

```bash
# 对比 wiki 概述 vs hpp @brief
SRC="/i/code/Prism/include/prism"
for f in $(find H:/PrismWiki -name '*.md' -not -path '*/.git/*' -not -path '*/.obsidian/*' -not -path '*/.plan/*' -not -path '*/skills/*' -not -path '*/ref/*' -not -path '*/dev/*' -not -path '*/client/*' -not -path '*/docs/*' -not -path '*/performance/*'); do
  src=$(head -15 "$f" | grep -m1 '^source:' | sed 's/^source: *"*//;s/"*$//')
  [ -z "$src" ] && continue
  hpp="${SRC}/../${src}"
  [ ! -f "$hpp" ] && continue
  hpp_brief=$(grep -m1 '@brief' "$hpp" | sed 's/.*@brief *//')
  wiki_title=$(head -5 "$f" | grep -m1 '^title:' | sed 's/^title: *"*//;s/"*$//')
  rel=$(echo "$f" | sed 's|H:/PrismWiki/||;s|\.md$||')
  [ -n "$hpp_brief" ] && echo "$rel  wiki=\"$wiki_title\"  hpp=\"$hpp_brief\""
done
```

---

## 批量操作

### 批量刷新已有文件

当需要按新标准刷新所有文件时：

1. **扫描差距**：运行审计脚本，生成缺失段落报告
2. **分优先级**：S 档 > A 档 > B 档；错误描述 > 缺失内容 > 格式问题
3. **分批处理**：每批 5-10 个文件，用 `delegate_task` 并行（最多 3 个）
4. **每批验证**：段落完整性、链接无死链
5. **更新 log.md**

### 批量生成新文件

当源码新增了多个 hpp 时：

1. 用 `execute_code` 扫描新增 hpp 列表
2. 确定每个 hpp 的档位
3. 按模块分批生成，每批 ~40 个文件（`write_file` 工具调用限制）
4. 生成后立即运行覆盖率审计
5. 更新 index.md

---

## 常见错误（Pitfalls）

### 内容错误

1. **不读源码就写文档**：hpp 的 `@brief` 才是真相，旧文档可能过时。**必须**读 hpp 再写。
2. **概述与 @brief 不一致**：wiki 说"连接器"但 hpp 说"Socket 异步 IO 适配器" → 概述是错的。以 hpp 为准。
3. **调用链凭记忆写**：必须 grep 整个项目确认。`grep -rn "函数名" include/prism/ src/prism/ --include="*.hpp" --include="*.cpp"`
4. **把外部库说成自己的**："smux 是 Prism 的协议" → 错。"Prism 实现了 smux（原作者 xtaci）" → 对。
5. **过于绝对的描述**："smux 只能配合 Trojan" → 错。smux 是通用协议。

### 结构错误

6. **module 字段用 C++ 命名空间**：`module: "stealth::shadowtls"` → 错。应为 `module: "stealth"`。
7. **source 字段标行号**：`source: "include/prism/xxx.hpp:42"` → 不要行号，只标路径。
8. **创建 _index.md**：用户明确不要。目录内直接放内容文件。
9. **空目录**：新建目录时必须至少有一个 md 文件。
10. **新建文件不更新 index.md**：每次新建都必须同步更新 index.md。

### 链接错误

11. **死链**：写 wikilink 前先确认目标文件存在。
12. **孤儿页**：每个页面至少被 index.md 链接一次。
13. **C++ 属性误判为 wikilink**：`[[nodiscard]]` 不是链接，grep 时要过滤。
14. **related 字段用 C++ 命名空间**：`related: ["stealth::shadowtls"]` → 错。应为 `related: ["[[core/stealth/shadowtls]]"]`。

### 格式错误

15. **贴源码片段**：不要在文档里贴代码块。用函数名 + 描述即可。
16. **英文描述**：所有描述文字用中文，代码标识符保持英文。
17. **frontmatter 日期格式**：`YYYY-MM-DD`，不要用其他格式。
18. **tags 数量**：3-6 个，不要太多也不要太少。

### 操作错误

19. **删除文件不清理引用**：必须 grep 找所有引用并清理。
20. **移动文件不批量更新链接**：必须用 `replace_all=true` 批量替换。
21. **刷新时不重读源码**：旧文档可能已过时，不能只在旧文档上修补。
22. **B 档文件强行凑字数**：简单文件就写简单，50 行够了就 50 行。
23. **S 档文件偷懒**：核心模块每个函数都要展开，不能用"类似，省略"。

---

## 完整模板

见 `references/templates.md`，包含：
- API 文档完整模板（S/A/B 三档各一个）
- 模块概览模板
- 协议文档模板
- 知识页模板
- 开发指南模板
- Bug 记录模板
- 性能报告模板
- 根目录基础设施文件模板
