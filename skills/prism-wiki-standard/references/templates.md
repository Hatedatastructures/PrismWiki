---
title: templates
created: 2026-05-18
updated: 2026-05-18
layer: skills
---

# PrismWiki 完整模板集

所有模板的 YAML frontmatter 部分用 `---` 包裹，正文紧跟其后。

---

## 1. API 文档模板（S 档 — 核心模块）

适用：stealth/shadowtls, crypto/aead, protocol 各 relay, agent/session, channel/transport/snapshot 等

```markdown
---
title: "文件名.hpp — 一句话中文描述"
source: "include/prism/模块/文件名.hpp"
module: "模块名"
type: api
tags: [模块, 子模块, 功能, 关键词1, 关键词2]
created: YYYY-MM-DD
updated: YYYY-MM-DD
related:
  - "[[模块/相关文件1]]"
  - "[[模块/相关文件2]]"
  - "[[ref/相关知识]]"
---

# 文件名.hpp

> 源码: `include/prism/模块/文件名.hpp`
> 实现: `src/prism/模块/文件名.cpp`
> 模块: [[模块|模块]] > [[模块/子模块|子模块]]

## 概述

一段话描述文件职责、设计意图、在系统中的角色。必须与 hpp @brief 一致。

## 命名空间

`psm::模块::子模块`

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[core/crypto/aead]] | 使用 AEAD 加解密 |
| 继承 | [[core/channel/transport/transmission]] | 传输层接口 |
| 被依赖 | [[core/agent/session/session|session]] | session 启动时调用 |

## 类与结构体

### 类: ClassName

一句话说明用途。继承自 [[base_class]]（如有）。

**成员变量**:
| 类型 | 名称 | 说明 |
|------|------|------|
| type | member_ | 描述 |
| config const& | config_ | 配置引用 |

**关键方法**:
- [[#函数: method1]] — 说明
- [[#函数: method2]] — 说明

## 配置项

|| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| timeout_ms | uint32_t | 30000 | 超时时间 |
| max_retries | int | 3 | 最大重试次数 |

## 函数签名表

|| 函数 | 签名 | 说明 |
|------|------|------|------|
| [[#函数: method1]] | `auto method1(type param) -> ret` | 说明 |
| [[#函数: method2]] | `auto method2(type param) -> ret` | 说明 |

## 函数: method1

> 源码: `include/prism/模块/文件名.hpp:XX`

**功能**: 一句话描述函数做什么

**签名**:
`auto method1(config const& cfg) -> bool`

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| cfg | config const& | 配置引用 |

**返回值**: `bool` — 成功返回 true，失败返回 false

**调用链**:
- **调用**（向下）:
  - [[core/crypto/aead#函数: encrypt]] — 加密握手数据
  - [[core/protocol/common/address#函数: parse]] — 解析目标地址
- **被调用**（向上）:
  - [[core/agent/session/session#函数: start|session#函数: start]] — session 启动时调用

**知识域**（可选）:
- [[ref/crypto/tls-handshake]] — TLS 握手流程
- [[ref/protocol/shadowsocks]] — SS 协议规范

## 函数: method2

**功能**: 一句话描述

**签名**:
`auto method2(buffer_view data) -> result`

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| data | buffer_view | 输入数据 |

**返回值**: `result` — 描述

**调用链**:
- **调用**（向下）:
  - [[#函数: internal_helper]] — 内部辅助函数
- **被调用**（向上）:
  - [[core/stealth/scheme#函数: try_scheme]] — 伪装方案尝试
```

---

## 2. API 文档模板（A 档 — 标准模块）

适用：protocol 各 config/constants/format, multiplex/smux|yamux, resolve/dns 子模块

```markdown
---
title: "config.hpp — yamux 协议配置"
source: "include/prism/multiplex/yamux/config.hpp"
module: "multiplex"
type: api
tags: [multiplex, yamux, config, 配置]
created: YYYY-MM-DD
updated: YYYY-MM-DD
related:
  - "[[core/multiplex/config]]"
  - "[[core/multiplex/yamux/craft]]"
---

# config.hpp — yamux 协议配置

> 源码: `include/prism/multiplex/yamux/config.hpp`
> 模块: [[multiplex|multiplex]] > yamux

## 概述

yamux 协议配置。定义 yamux 协议的全部配置参数，包括流数量限制、流量控制窗口、心跳和超时设置。

## 命名空间

`psm::multiplex::yamux`

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[core/multiplex/yamux/craft]] | 帧构造使用配置参数 |
| 被依赖 | [[core/multiplex/core]] | 核心层加载配置 |

## 类与结构体

### 类: config

yamux 协议配置结构体。

**成员变量**:
| 类型 | 名称 | 说明 |
|------|------|------|
| uint32_t | max_stream | 最大并发流数 |
| uint32_t | initial_window | 初始窗口大小 |
| bool | enable_ping | 是否启用心跳 |

## 配置项

|| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| max_stream | uint32_t | 256 | 最大并发流数 |
| initial_window | uint32_t | 262144 | 初始窗口大小（256KB） |
| enable_ping | bool | true | 是否启用心跳 |
| ping_interval_ms | uint32_t | 30000 | 心跳间隔 |

## 函数签名表

|| 函数 | 签名 | 说明 |
|------|------|------|------|
| （无公开函数） | — | 通过 glaze 反序列化 |
```

---

## 3. API 文档模板（B 档 — 简单模块）

适用：base64, sha224, block, constants 文件

```markdown
---
title: "base64 — Base64 编解码"
source: "include/prism/crypto/base64.hpp"
module: "crypto"
type: api
tags: [crypto, base64, 编码]
created: YYYY-MM-DD
updated: YYYY-MM-DD
related:
  - "[[core/crypto/aead]]"
---

# base64.hpp

> 源码: `include/prism/crypto/base64.hpp`
> 实现: `src/prism/crypto/base64.cpp`
> 模块: [[crypto|crypto]]

## 概述

Base64 编解码工具。提供标准 Base64 和 URL-safe Base64 的编码和解码功能。

## 命名空间

`psm::crypto`

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 被依赖 | [[core/stealth/reality/auth]] | Reality 认证数据编码 |
| 被依赖 | [[core/protocol/trojan/relay]] | Trojan 凭据编码 |

## 函数签名表

|| 函数 | 签名 | 说明 |
|------|------|------|------|
| [[#函数: encode]] | `auto encode(span<const uint8_t>) -> string` | Base64 编码 |
| [[#函数: decode]] | `auto decode(string_view) -> vector<uint8_t>` | Base64 解码 |
| [[#函数: encode_url]] | `auto encode_url(span<const uint8_t>) -> string` | URL-safe 编码 |

## 函数: encode

**功能**: 将二进制数据编码为标准 Base64 字符串

**签名**: `auto encode(span<const uint8_t> data) -> string`

**参数**: `data` — 待编码的二进制数据

**返回值**: `string` — Base64 编码字符串

**调用链**:
- **调用**: 无外部依赖
- **被调用**: [[core/stealth/reality/auth#函数: sign]]

## 函数: decode

**功能**: 将 Base64 字符串解码为二进制数据

**签名**: `auto decode(string_view encoded) -> vector<uint8_t>`

**参数**: `encoded` — Base64 编码字符串

**返回值**: `vector<uint8_t>` — 解码后的二进制数据

**调用链**:
- **调用**: 无外部依赖
- **被调用**: [[core/protocol/trojan/relay#函数: verify_credential]]
```

---

## 4. 模块概览模板（多 hpp 合并）

适用：loader.md, transformer.md, memory.md, outbound.md 等基础设施模块

```markdown
---
title: "Loader 模块"
module: "loader"
type: module
tags: [loader, load, 配置, json, glaze]
created: YYYY-MM-DD
updated: YYYY-MM-DD
related:
  - "[[core/transformer/json]]"
  - "[[core/agent/account/directory|directory]]"
---

# Loader 模块

> 模块: [[loader|loader]]

## 概述

Loader 模块负责配置文件加载和账户目录构建。所有函数为 header-only 内联实现。

---

## load.hpp — 配置加载适配器

> 源码: `include/prism/loader/load.hpp`
> 模块: [[loader|loader]]

### 概述

配置加载适配器。将外部 JSON 配置转换为内部 `psm::config` 结构。

### 命名空间

`psm::loader`

### 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[core/transformer/json]] | glaze JSON 反序列化 |
| 被依赖 | `main.cpp` | 启动时加载配置 |

### 函数: load()

- **功能**: 从文件系统加载配置文件
- **签名**: `inline auto load(std::string_view path) -> config`
- **参数**: `path` — 配置文件路径
- **返回值**: `config` — 配置对象
- **调用（向下）**: `transformer::json::deserialize()`
- **被调用（向上）**: `main.cpp` 启动流程

---

## build.hpp — 账户目录构建

> 源码: `include/prism/loader/build.hpp`
> 模块: [[loader|loader]]

### 概述

从认证配置构建账户目录。

### 函数: build_account_directory()

- **功能**: 将认证配置转换为运行时账户目录
- **签名**: `inline auto build_account_directory(const agent::authentication &auth) -> shared_ptr<agent::account::directory>`
- **参数**: `auth` — 认证配置
- **返回值**: 账户目录智能指针
- **调用（向下）**: `crypto::normalize_credential()`, `account::directory::insert()`
- **被调用（向上）**: `main.cpp` 启动流程

---

## 相关页面

- [[core/transformer/json]] — JSON 序列化
- [[core/agent/account/directory|directory]] — 账户目录
```

---

## 5. 协议文档模板（docs/）

适用：docs/stealth/shadowtls.md, docs/protocol/trojan.md 等

```markdown
---
title: ShadowTLS
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: protocol
tags: [stealth, tls, hmac, tier1]
related:
  - "[[core/stealth/shadowtls/auth]]"
  - "[[core/stealth/shadowtls/handshake]]"
  - "[[docs/stealth/restls]]"
---

# ShadowTLS

一句话概述。

## 工作原理

详细技术说明...

## 版本差异

### v2
...

### v3
...

## 与 Prism 实现的关系

Prism 在 stealth 模块中实现了 Tier 1 级别的兼容版本。
相关实现文件：[[core/stealth/shadowtls/auth|auth]]、[[core/stealth/shadowtls/handshake|handshake]]

## 参考资料

- [GitHub - ShadowTLS](https://github.com/...)

## 相关页面

- [[core/stealth/shadowtls/auth]] — 认证实现
- [[core/stealth/shadowtls/handshake]] — 握手实现
```

---

## 6. 知识页模板（ref/）

适用：ref/crypto/hmac-sha1.md, ref/protocol/tls-1.3.md 等

```markdown
---
title: "HMAC-SHA1"
category: "crypto"
type: ref
tags: [密码学, hmac, sha1, 消息认证码]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# HMAC-SHA1

**类别**: 密码学

## 概述

HMAC-SHA1 是一种基于 SHA-1 的消息认证码算法，用于验证消息的完整性和真实性。

## 原理

### HMAC 结构

详细算法说明...

**算法步骤**：
1. ...
2. ...

### 安全性

安全性分析...

## 在 Prism 中的应用

| 函数/类 | 使用方式 | 链接 |
|---------|----------|------|
| verify_client_hello | 验证 SessionID | [[core/stealth/shadowtls/auth|auth]] |
| compute_hmac | 计算 HMAC 标签 | [[core/stealth/shadowtls/auth|auth]] |

## 参考资料

- [RFC 2104](https://tools.ietf.org/html/rfc2104)

## 相关知识

- [[ref/crypto/hmac-sha256]] — 更安全的变体
- [[ref/protocol/tls-sessionid]] — TLS SessionID
```

---

## 7. 开发指南模板（dev/）

适用：dev/testing.md, dev/cpp23-coroutines.md 等

```markdown
---
title: 测试体系
tags: [testing, ctest, unit, e2e, regression]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# 测试体系

概述...

## 运行方式

```bash
ctest --test-dir build_release --output-on-failure
```

## 测试分类

| 类别 | 测试文件 | 说明 |
|------|----------|------|
| 基础设施 | Fault.cpp, Memory.cpp | ... |
| 协议 | Socks5.cpp, Trojan.cpp | ... |

## 相关页面

- [[dev/building/configuration|configuration]] — 配置详解
- [[dev/modules]] — 模块结构
```

---

## 8. Bug 记录模板

```markdown
---
title: "ShadowTLS 握手超时"
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: bug
severity: major
modules: [stealth, shadowtls]
status: fixed
fix_commit: abc1234
---

# ShadowTLS 握手超时

## 现象

描述用户可见的症状...

## 排查过程

1. 第一步排查...
2. 第二步排查...

## 根因

根本原因分析...

## 修复

修复方案和代码位置...

## 相关模块

- [[core/stealth/shadowtls/handshake]]
- [[core/channel/transport/snapshot]]

## 预防措施

如何避免类似问题...
```

---

## 9. 性能报告模板（performance/）

适用：performance/benchmark.md, performance/report.md

```markdown
---
title: Prism 代理性能基准测试
type: performance
tags: [performance, benchmark, latency, throughput, proxy]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# Prism 代理性能基准测试

## 测试环境

| 项目 | 配置 |
|------|------|
| CPU | Intel i9-13900K |
| 内存 | 64GB DDR5 |
| OS | Windows 11 |
| 编译器 | MSVC 2022 / Clang 18 |
| 构建类型 | Release |

## 测试方法

测试工具、测试场景、并发参数...

## 结果数据

| 场景 | 吞吐量 | P50 延迟 | P99 延迟 |
|------|--------|----------|----------|
| TCP 直连 | ... | ... | ... |
| Trojan + TLS | ... | ... | ... |

## 分析结论

性能分析结论...

## 相关页面

- [[dev/modules]] — 模块结构
- [[core/channel/transport/reliable]] — TCP 传输层
```

---

## 10. 根目录基础设施文件模板

适用于：exception.md, fault.md, trace.md 等只有单个 hpp 的根目录文件

```yaml
---
title: "Exception 模块"
module: "exception"
type: module
tags: [exception, 异常, 错误处理]
created: YYYY-MM-DD
updated: YYYY-MM-DD
related:
  - "[[fault]]"
  - "[[memory]]"
---
```

正文结构与模块概览模板相同，如果只有一个 hpp 就按 API 文档模板写。
