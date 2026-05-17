# PrismWiki 知识库重构设计方案

> 创建时间: 2026-05-17
> 状态: 待用户审核

---

## 一、设计决策总结

### 1. 知识库分层

| 层级 | 目录 | 职责 | 详细程度 |
|------|------|------|----------|
| 模块实现层 | `core/` | 函数实现、调用链、状态变化 | 详细，复杂函数逐行解释 |
| 开发排障层 | `dev/` | 开发规范、排障方法、Bug记录 | 详细 |
| 使用指南层 | `docs/` | 用户文档、部署、配置 | 简单 |
| 参考知识层 | `ref/` | 协议规范、加密原理、mihomo资料 | 中等 |

### 2. 详细程度标准

- **详细层 (core/dev)**：
  - 复杂函数：逐行解释实现逻辑
  - 简单函数：写用途，不解释实现
  - 复杂判断：综合判断（代码行数、逻辑分支、知识点、状态变化）
  - 调用链：混合形式 `调用 [[module/file|函数]]()` 或仅 wikilink

- **中等层 (ref)**：
  - 协议帧格式、字节布局
  - 常量定义、枚举值含义
  - 配置项说明

- **简单层 (docs)**：
  - 一句话说明功能
  - 如何启用/配置
  - 常见问题一句话解答

### 3. 知识库与源码关系

- **Prism 源码位置**: `I:/code/Prism/`
- **Mihomo 源码位置**: `I:/code/Prism/logs/mihomo-Meta/`
- **不重复写知识**：同一知识点只在一个地方详细写，其他地方用 wikilink 引用
- **docs/prism/ vs Wiki**：docs/prism/ 保留简要说明，Wiki 是详细版本

### 4. Skill 设计

**wiki-sync skill** — 知识库同步检查

- **触发时机**：
  - Push: 全面检查
  - Release: 深度检查（仔细分析逻辑变化）
  - 手动: `/wiki-sync` 命令

- **输出格式**：
  - 严重问题 → 终端输出
  - 完整报告 → `.claude/wiki-sync-report.md`

- **检查项（普通）**：
  1. 源码路径是否存在
  2. 函数签名是否变化
  3. 调用链是否变化
  4. 枚举/常量是否变化
  5. 类结构变化（成员变量、成员函数、继承关系）
  6. 配置项变化（JSON 结构、字段名、默认值）
  7. 错误码变化
  8. 协议兼容性
  9. 测试覆盖
  10. 依赖关系变化
  11. 文档结构（新增代码是否有对应 wiki）

- **检查项（深度，Release时）**：
  1. 文档一致性（同一模块多个页面是否矛盾）
  2. 链接完整性（wikilink 目标是否存在）
  3. 覆盖度检查（源码目录 vs core/ 目录）
  4. 变更影响分析（函数改名后所有引用是否更新）
  5. 新功能检测（新增源码文件是否需要新建 wiki）

- **替换原有 skill**：
  - 替换 `bug-to-wiki`
  - 替换 `module-analysis`

---

## 二、完整目录结构

```
H:/PrismWiki/
├── core/                      # 模块实现细节（详细）
│   ├── agent/
│   ├── channel/
│   ├── pipeline/
│   ├── protocol/
│   ├── stealth/
│   ├── recognition/
│   ├── resolve/
│   ├── multiplex/
│   ├── crypto/
│   ├── outbound/
│   ├── memory/
│   ├── fault/
│   ├── exception/
│   ├── trace/
│   ├── transformer/
│   ├── loader/
│   ├── infrastructure.md
│   ├── architecture.md
│   ├── startup.md
│   └── flow.md
│
├── dev/                       # 开发规范、排障方法（详细）
│   ├── coding/
│   ├── testing/
│   ├── debugging/
│   ├── building/
│   ├── performance/
│   ├── extending/
│   ├── bugs/
│   ├── roadmap.md
│   └── overview.md
│
├── docs/                      # 使用指南（简单）
│   ├── getting-started.md
│   ├── deployment.md
│   ├── configuration.md
│   ├── client-setup.md
│   ├── faq.md
│   ├── troubleshooting.md
│   ├── upgrade.md
│   ├── security.md
│   ├── performance-tips.md
│   └── overview.md
│
├── ref/                       # 参考知识（中等）
│   ├── mihomo/                # 完整 mihomo 参考（见下方详细结构）
│   ├── crypto/
│   ├── protocol/
│   ├── network/
│   ├── programming/
│   ├── glossary.md
│   └── overview.md
│
├── skills/                    # Claude Code Skills
│   └ prism-wiki-sync/
│   │   ├── SKILL.md           # Skill 定义
│   │   ├── checks.md          # 检查项详细说明
│   │   ├── report-template.md # 报告模板
│   │   └── references/        # 参考文档
│   │       ├── prism-structure.md  # Prism 源码结构参考
│   │       └ wiki-structure.md     # Wiki 目录结构参考
│   │
├── SCHEMA.md                  # 知识库规范
├── index.md                   # 总索引
└── log.md                     # 操作日志
```

### core/ 详细结构

```
core/
├── agent/
│   ├── front/
│   │   ├── listener.md        # 源码: include/prism/agent/front/listener.hpp
│   │   └── balancer.md        # 源码: include/prism/agent/front/balancer.hpp
│   ├── session/
│   │   └── session.md         # 源码: include/prism/agent/session/session.hpp
│   ├── worker/
│   │   ├── worker.md          # 源码: include/prism/agent/worker/worker.hpp
│   │   ├── launch.md          # 源码: include/prism/agent/worker/launch.hpp
│   │   ├── stats.md           # 源码: include/prism/agent/worker/stats.hpp
│   │   └── tls.md             # 源码: include/prism/agent/worker/tls.hpp
│   ├── account/
│   │   ├── entry.md           # 源码: include/prism/agent/account/entry.hpp
│   │   └── directory.md       # 源码: include/prism/agent/account/directory.hpp
│   ├── dispatch/
│   │   └ table.md             # 源码: include/prism/agent/dispatch/table.hpp
│   ├── config.md              # 源码: include/prism/agent/config.hpp
│   ├── context.md             # 源码: include/prism/agent/context.hpp
│   └── overview.md
│
├── channel/
│   ├── transport/
│   │   ├── transmission.md    # 源码: include/prism/channel/transport/transmission.hpp
│   │   ├── reliable.md        # 源码: include/prism/channel/transport/reliable.hpp
│   │   ├── encrypted.md       # 源码: include/prism/channel/transport/encrypted.hpp
│   │   ├── unreliable.md      # 源码: include/prism/channel/transport/unreliable.hpp
│   │   └ snapshot.md          # 源码: include/prism/channel/transport/snapshot.hpp (如存在)
│   ├── connection/
│   │   └ pool.md              # 源码: include/prism/channel/connection/pool.hpp
│   ├── eyeball/
│   │    racer.md              # 源码: include/prism/channel/eyeball/racer.hpp
│   ├── health.md              # 源码: include/prism/channel/health.hpp
│   ├── adapter/
│   │   └ connector.md         # 源码: include/prism/channel/adapter/connector.hpp
│   └ overview.md
│
├── pipeline/
│   ├── primitives.md          # 源码: include/prism/pipeline/primitives.hpp
│   ├── protocols/
│   │   ├── http.md            # 源码: include/prism/pipeline/protocols/http.hpp
│   │   ├── socks5.md          # 源码: include/prism/pipeline/protocols/socks5.hpp
│   │   ├── trojan.md          # 源码: include/prism/pipeline/protocols/trojan.hpp
│   │   ├── vless.md           # 源码: include/prism/pipeline/protocols/vless.hpp
│   │   └ shadowsocks.md       # 源码: include/prism/pipeline/protocols/shadowsocks.hpp
│   └ overview.md
│
├── protocol/
│   ├── common/
│   │   ├── address.md         # 源码: include/prism/protocol/common/address.hpp
│   │   ├── form.md            # 源码: include/prism/protocol/common/form.hpp
│   │   └ read.md              # 源码: include/prism/protocol/common/read.hpp
│   ├── http/
│   │   ├── parser.md          # 源码: include/prism/protocol/http/parser.hpp
│   │    relay.md              # 源码: include/prism/protocol/http/relay.hpp
│   ├── socks5/
│   │   ├── wire.md            # 源码: include/prism/protocol/socks5/wire.hpp
│   │   ├── stream.md          # 源码: include/prism/protocol/socks5/stream.hpp
│   │   ├── constants.md       # 源码: include/prism/protocol/socks5/constants.hpp
│   │   ├── config.md          # 源码: include/prism/protocol/socks5/config.hpp
│   ├── trojan/
│   │   ├── format.md          # 源码: include/prism/protocol/trojan/format.hpp
│   │   ├── relay.md           # 源码: include/prism/protocol/trojan/relay.hpp
│   │   ├── constants.md       # 源码: include/prism/protocol/trojan/constants.hpp
│   │   ├── config.md          # 源码: include/prism/protocol/trojan/config.hpp
│   ├── vless/
│   │   ├── format.md          # 源码: include/prism/protocol/vless/format.hpp
│   │   ├── relay.md           # 源码: include/prism/protocol/vless/relay.hpp
│   │   ├── constants.md       # 源码: include/prism/protocol/vless/constants.hpp
│   │   ├── config.md          # 源码: include/prism/protocol/vless/config.hpp
│   ├── shadowsocks/
│   │   ├── format.md          # 源码: include/prism/protocol/shadowsocks/format.hpp
│   │   ├── relay.md           # 源码: include/prism/protocol/shadowsocks/relay.hpp
│   │   ├── salts.md           # 源码: include/prism/protocol/shadowsocks/salts.hpp
│   │   ├── replay.md          # 源码: include/prism/protocol/shadowsocks/replay.hpp
│   │   ├── constants.md       # 源码: include/prism/protocol/shadowsocks/constants.hpp
│   │   ├── config.md          # 源码: include/prism/protocol/shadowsocks/config.hpp
│   ├── tls/
│   │   ├── types.md           # 源码: include/prism/protocol/tls/types.hpp
│   │   ├── signal.md          # 源码: include/prism/protocol/tls/signal.hpp
│   │   ├── feature_bitmap.md  # 源码: include/prism/protocol/tls/feature_bitmap.hpp
│   ├── analysis.md            # 源码: include/prism/protocol/analysis.hpp
│   └ overview.md
│
├── stealth/
│   ├── scheme.md              # 源码: include/prism/stealth/scheme.hpp
│   ├── executor.md            # 源码: include/prism/stealth/executor.hpp
│   ├── registry.md            # 源码: include/prism/stealth/registry.hpp
│   ├── native.md              # 源码: include/prism/stealth/native.hpp
│   ├── reality/
│   │   ├── scheme.md          # 源码: include/prism/stealth/reality/scheme.hpp, src/prism/stealth/reality/scheme.cpp
│   │   ├── handshake.md       # 源码: include/prism/stealth/reality/handshake.hpp, src/prism/stealth/reality/handshake.cpp
│   │   ├── auth.md            # 源码: include/prism/stealth/reality/auth.hpp
│   │   ├── keygen.md          # 源码: include/prism/stealth/reality/keygen.hpp
│   │   ├── request.md         # 源码: include/prism/stealth/reality/request.hpp
│   │   ├── response.md        # 源码: include/prism/stealth/reality/response.hpp
│   │   ├── seal.md            # 源码: include/prism/stealth/reality/seal.hpp
│   │   ├── config.md          # 源码: include/prism/stealth/reality/config.hpp
│   │   ├── constants.md       # 源码: include/prism/stealth/reality/constants.hpp
│   ├── shadowtls/
│   │   ├── scheme.md          # 源码: include/prism/stealth/shadowtls/scheme.hpp
│   │   ├── handshake.md       # 源码: include/prism/stealth/shadowtls/handshake.hpp
│   │   ├── auth.md            # 源码: include/prism/stealth/shadowtls/auth.hpp
│   │   ├── config.md          # 源码: include/prism/stealth/shadowtls/config.hpp
│   │   ├── constants.md       # 源码: include/prism/stealth/shadowtls/constants.hpp
│   ├── restls/
│   │   ├── scheme.md          # 源码: include/prism/stealth/restls/scheme.hpp
│   │   ├── config.md          # 源码: include/prism/stealth/restls/config.hpp
│   ├── anytls/
│   │   ├── scheme.md          # 源码: include/prism/stealth/anytls/scheme.hpp
│   │   ├── config.md          # 源码: include/prism/stealth/anytls/config.hpp
│   ├── trusttunnel/
│   │   ├── scheme.md          # 源码: include/prism/stealth/trusttunnel/scheme.hpp
│   │   ├── config.md          # 源码: include/prism/stealth/trusttunnel/config.hpp
│   ├── ech/
│   │   ├── decrypt.md         # 源码: include/prism/stealth/ech/decrypt.hpp
│   │   ├── config.md          # 源码: include/prism/stealth/ech/config.hpp
│   └ overview.md
│
├── recognition/
│   ├── recognition.md         # 源码: include/prism/recognition/recognition.hpp
│   ├── result.md              # 源码: include/prism/recognition/result.hpp
│   ├── confidence.md          # 源码: include/prism/recognition/confidence.hpp
│   ├── layered_pipeline.md    # 源码: include/prism/recognition/layered_pipeline.hpp
│   ├── probe/
│   │   ├── probe.md           # 源码: include/prism/recognition/probe/probe.hpp
│   │   ├── analyzer.md        # 源码: include/prism/recognition/probe/analyzer.hpp
│   │   └─ scheme_route_table.md # 源码: include/prism/recognition/scheme_route_table.hpp
│   └ overview.md
│
├── resolve/
│   ├── router.md              # 源码: include/prism/resolve/router.hpp, src/prism/resolve/router.cpp
│   ├── dns/
│   │   ├── dns.md             # 源码: include/prism/resolve/dns/dns.hpp
│   │   ├── upstream.md        # 源码: include/prism/resolve/dns/upstream.hpp
│   │   ├── config.md          # 源码: include/prism/resolve/dns/config.hpp
│   │   ├── detail/
│   │   │   ├── cache.md       # 源码: include/prism/resolve/dns/detail/cache.hpp
│   │   │   ├── coalescer.md   # 源码: include/prism/resolve/dns/detail/coalescer.hpp
│   │   │   ├── rules.md       # 源码: include/prism/resolve/dns/detail/rules.hpp
│   │   │   ├── format.md      # 源码: include/prism/resolve/dns/detail/format.hpp
│   │   │   └─ utility.md      # 源码: include/prism/resolve/dns/detail/utility.hpp
│   └ overview.md
│
├── multiplex/
│   ├── core.md                # 源码: include/prism/multiplex/core.hpp
│   ├── bootstrap.md           # 源码: include/prism/multiplex/bootstrap.hpp
│   ├── duct.md                # 源码: include/prism/multiplex/duct.hpp
│   ├── parcel.md              # 源码: include/prism/multiplex/parcel.hpp
│   ├── config.md              # 源码: include/prism/multiplex/config.hpp
│   ├── smux/
│   │   ├── craft.md           # 源码: include/prism/multiplex/smux/craft.hpp
│   │   ├── frame.md           # 源码: include/prism/multiplex/smux/frame.hpp
│   │   ├── config.md          # 源码: include/prism/multiplex/smux/config.hpp
│   ├── yamux/
│   │   ├── craft.md           # 源码: include/prism/multiplex/yamux/craft.hpp
│   │   ├── frame.md           # 源码: include/prism/multiplex/yamux/frame.hpp
│   │   ├── config.md          # 源码: include/prism/multiplex/yamux/config.hpp
│   └ overview.md
│
├── crypto/
│   ├── aead.md                # 源码: include/prism/crypto/aead.hpp, src/prism/crypto/aead.cpp
│   ├── hkdf.md                # 源码: include/prism/crypto/hkdf.hpp
│   ├── x25519.md              # 源码: include/prism/crypto/x25519.hpp
│   ├── blake3.md              # 源码: include/prism/crypto/blake3.hpp
│   ├── block.md               # 源码: include/prism/crypto/block.hpp
│   ├── base64.md              # 源码: include/prism/crypto/base64.hpp
│   ├── sha224.md              # 源码: include/prism/crypto/sha224.hpp (如存在)
│   └ overview.md
│
├── outbound/
│   ├── proxy.md               # 源码: include/prism/outbound/proxy.hpp
│   ├── direct.md              # 源码: include/prism/outbound/direct.hpp
│   └ overview.md
│
├── memory/
│   ├── container.md           # 源码: include/prism/memory/container.hpp
│   ├── pool.md                # 源码: include/prism/memory/pool.hpp
│   └ overview.md
│
├── fault/
│   ├── code.md                # 源码: include/prism/fault/code.hpp
│   ├── handling.md            # 源码: include/prism/fault/handling.hpp
│   ├── compatible.md          # 源码: include/prism/fault/compatible.hpp
│   └ overview.md
│
├── exception/
│   ├── deviant.md             # 源码: include/prism/exception/deviant.hpp
│   ├── network.md             # 源码: include/prism/exception/network.hpp
│   ├── protocol.md            # 源码: include/prism/exception/protocol.hpp
│   ├── security.md            # 源码: include/prism/exception/security.hpp
│   └ overview.md
│
├── trace/
│   ├── config.md              # 源码: include/prism/trace/config.hpp
│   ├── spdlog.md              # 源码: src/prism/trace/spdlog.cpp (实现细节)
│   └ overview.md
│
├── transformer/
│   ├── json.md                # 源码: include/prism/transformer/json.hpp
│   └ overview.md
│
├── loader/
│   ├── load.md                # 源码: include/prism/loader/load.hpp
│   └ overview.md
│
├── infrastructure.md          # 基础设施总览（memory/fault/exception/trace/transformer/loader）
├── architecture.md            # 整体架构总览（六阶段流水线）
├── startup.md                 # 启动流程详解（main.cpp 9步）
├── flow.md                    # 协议处理流程详解
└── overview.md
```

### dev/ 详细结构

```
dev/
├── coding/
│   ├── naming.md              # 命名规范（CLAUDE.md §命名与编码规范）
│   ├── coroutine.md           # 协程约定（CLAUDE.md §协程约定、§协程纯度要求）
│   ├── pmr.md                 # PMR 使用规范（CLAUDE.md §PMR内存策略）
│   ├── doxygen.md             # 注释规范（CLAUDE.md §命名与编码规范-注释）
│   ├── lifecycle.md           # 生命周期安全（CLAUDE.md §生命周期安全）
│   ├── error.md               # 错误处理策略（CLAUDE.md §错误处理）
│   └ overview.md
│
├── testing/
│   ├── framework.md           # 测试框架（CLAUDE.md §测试）
│   ├── writing.md             # 测试编写指南
│   ├── commands.md            # 测试命令（CLAUDE.md §构建命令-运行测试）
│   ├── benchmark.md           # 基准测试（CLAUDE.md §构建命令-运行基准测试）
│   ├── stress.md              # 压力测试（CLAUDE.md §构建命令-运行压力测试）
│   ├── concurrency.md         # 并发测试（CLAUDE.md §并发测试）
│   └ overview.md
│
├── debugging/
│   ├── connection.md          # 连接问题排查
│   ├── protocol.md            # 协议问题排查
│   ├── memory.md              # 内存问题排查（PMR相关）
│   ├── performance.md         # 性能问题排查
│   ├── tls.md                 # TLS 问题排查
│   ├── log-analysis.md        # 日志分析方法
│   ├── common-issues.md       # 常见问题汇总
│   └ overview.md
│
├── building/
│   ├── cmake.md               # CMake 结构（CLAUDE.md §构建系统结构）
│   ├── dependencies.md        # 依赖管理（CLAUDE.md §依赖项）
│   ├── commands.md            # 构建命令（CLAUDE.md §构建命令）
│   ├── options.md             # 构建选项（CLAUDE.md §构建系统结构-构建选项）
│   └ overview.md
│
├── performance/
│   ├── tuning.md              # 调优方法
│   ├── profiling.md           # 性能分析
│   ├── report.md              # 性能报告（docs/prism/performance-report.md）
│   └ overview.md
│
├── extending/
│   ├── protocol.md            # 新协议集成（.claude/skills/protocol-handler）
│   ├── stealth.md             # 新伪装方案（CLAUDE.md §Recognition模块-插件架构）
│   ├── module.md              # 新模块开发
│   └ overview.md
│
├── bugs/
│   ├── template.md            # Bug 报告模板
│   └─ *.md                    # 各 Bug 详细记录
│
├── roadmap.md                 # 开发路线图（CLAUDE.md §开发路线图、plan.md）
└── overview.md
```

### docs/ 详细结构

```
docs/
├── getting-started.md         # 快速开始（docs/tutorial/getting-started.md）
├── deployment.md              # 部署指南（docs/tutorial/deployment.md）
├── configuration.md           # 配置说明（简化版，参考 configuration.json）
├── client-setup.md            # 客户端配置（docs/clash/config.yaml）
├── faq.md                     # 常见问题（docs/tutorial/faq.md）
├── troubleshooting.md         # 故障排查（简化版，docs/tutorial/troubleshooting.md）
├── upgrade.md                 # 升级指南
├── security.md                # 安全注意事项
├── performance-tips.md        # 性能建议
└── overview.md
```

### ref/ 详细结构（含完整 mihomo）

```
ref/
├── mihomo/
│   ├── overview.md            # mihomo 概述
│   │
│   ├── protocols/             # 出站协议（源码: adapter/outbound/*.go）
│   │   ├── socks5.md          # 源码: adapter/outbound/socks5.go
│   │   ├── socks4.md          # 源码: transport/socks4/
│   │   ├── http.md            # 源码: adapter/outbound/http.go
│   │   ├── trojan.md          # 源码: adapter/outbound/trojan.go, transport/trojan/
│   │   ├── vless.md           # 源码: adapter/outbound/vless.go, transport/vless/
│   │   ├── vmess.md           # 源码: adapter/outbound/vmess.go, transport/vmess/
│   │   ├── shadowsocks.md     # 源码: adapter/outbound/shadowsocks.go, transport/shadowsocks/
│   │   ├── shadowsocksr.md    # 源码: adapter/outbound/shadowsocksr.go, transport/ssr/
│   │   ├── snell.md           # 源码: adapter/outbound/snell.go, transport/snell/
│   │   ├── ssh.md             # 源码: adapter/outbound/ssh.go
│   │   ├── hysteria.md        # 源码: adapter/outbound/hysteria.go, transport/hysteria/
│   │   ├── hysteria2.md       # 源码: adapter/outbound/hysteria2.go
│   │   ├── tuic.md            # 源码: adapter/outbound/tuic.go
│   │   ├── wireguard.md       # 源码: adapter/outbound/wireguard.go
│   │   ├── anytls.md          # 源码: adapter/outbound/anytls.go, transport/anytls/
│   │   ├── reality.md         # 源码: adapter/outbound/reality.go
│   │   ├── ech.md             # 源码: adapter/outbound/ech.go
│   │   ├── trusttunnel.md     # 源码: adapter/outbound/trusttunnel.go, transport/trusttunnel/
│   │   ├── mieru.md           # 源码: adapter/outbound/mieru.go
│   │   ├── masque.md          # 源码: adapter/outbound/masque.go, transport/masque/
│   │   ├── sudoku.md          # 源码: adapter/outbound/sudoku.go, transport/sudoku/
│   │   ├── direct.md          # 源码: adapter/outbound/direct.go
│   │   ├── reject.md          # 源码: adapter/outbound/reject.go
│   │   ├── dns.md             # 源码: adapter/outbound/dns.go
│   │   └ overview.md
│   │
│   ├── transport/             # 传输层插件（源码: transport/*/）
│   │   ├── shadowtls.md       # 源码: transport/shadowtls/, transport/sing-shadowtls/
│   │   ├── restls.md          # 源码: transport/restls/
│   │   ├── v2ray-plugin.md    # 源码: transport/v2ray-plugin/
│   │   ├── simple-obfs.md     # 源码: transport/simple-obfs/
│   │   ├── gost-plugin.md     # 源码: transport/gost-plugin/
│   │   ├── kcptun.md          # 源码: transport/kcptun/
│   │   ├── gun.md             # 源码: transport/gun/ (gRPC)
│   │   └ overview.md
│   │
│   ├── mux/                   # 多路复用
│   │   ├── smux.md            # 配置参数
│   │   ├── yamux.md           # 配置参数
│   │   ├── singmux.md         # 源码: adapter/outbound/singmux.go
│   │   ├── config.md          # mux 配置参数详解
│   │   └ overview.md
│   │
│   ├── proxy-groups/          # 代理组（源码: adapter/outboundgroup/*.go）
│   │   ├── selector.md        # 源码: adapter/outboundgroup/selector.go
│   │   ├── url-test.md        # 源码: adapter/outboundgroup/urltest.go
│   │   ├── fallback.md        # 源码: adapter/outboundgroup/fallback.go
│   │   ├── load-balance.md    # 源码: adapter/outboundgroup/loadbalance.go
│   │   ├── relay.md           # 链式代理（配置）
│   │   ├── config.md          # 代理组配置参数
│   │   └ overview.md
│   │
│   ├── rules/                 # 规则系统（源码: rules/*.go）
│   │   ├── domain.md          # 域名规则
│   │   ├── ipcidr.md          # IP CIDR 规则
│   │   ├── port.md            # 端口规则
│   │   ├── process.md         # 进程规则
│   │   ├── geosite.md         # GeoSite 规则集
│   │   ├── geoip.md           # GeoIP 规则集
│   │   ├── rule-set.md        # Rule Provider
│   │   ├── final.md           # 最终规则
│   │   ├── logic.md           # 逻辑规则 (AND/OR/NOT)
│   │   └ overview.md
│   │
│   ├── dns/                   # DNS 配置（源码: dns/*.go）
│   │   ├── servers.md         # DNS 服务器配置
│   │   ├── enable.md          # DNS 功能开关
│   │   ├── listen.md          # DNS 监听端口
│   │   ├── enhanced-mode.md   # fake-ip / redir-host
│   │   ├── fake-ip-filter.md  # fake-ip 过滤
│   │   ├── nameserver-policy.md
│   │   ├── fallback.md        # fallback DNS
│   │   ├── fallback-filter.md
│   │   └ overview.md
│   │
│   ├── tun/                   # TUN 模式
│   │   ├── enable.md          # TUN 开关
│   │   ├── stack.md           # system/gvisor/lwip
│   │   ├── dns-hijack.md      # DNS 劫持
│   │   ├── auto-route.md      # 自动路由
│   │   ├── auto-detect-interface.md
│   │   └ overview.md
│   │
│   ├── config/                # 配置模板（源码: config/*.go, docs/）
│   │   ├── basic.yaml         # 基础配置模板
│   │   ├── advanced.yaml      # 高级配置模板
│   │   ├── mux.yaml           # mux 配置模板
│   │   ├── tun.yaml           # TUN 配置模板
│   │   ├── dns.yaml           # DNS 配置模板
│   │   ├── rules.yaml         # 规则配置模板
│   │   ├── proxy-groups.yaml  # 代理组配置模板
│   │   ├── full-example.yaml  # 完整示例
│   │   └ overview.md
│   │
│   ├── compatibility/         # 兼容性
│   │   ├── prism.md           # Prism 兼容性对照（哪些协议互通）
│   │   ├── features.md        # 功能支持矩阵
│   │   ├── protocol-matrix.md # 协议支持矩阵（mihomo支持 vs Prism支持）
│   │   ├── transport-matrix.md
│   │   └ overview.md
│   │
│   ├── implementation/        # 实现参考
│   │   ├── udp-relay.md       # UDP 中继实现方式
│   │   ├── tcp-concurrent.md  # TCP 并发连接
│   │   ├── keepalive.md       # Keepalive 机制
│   │   ├── authentication.md  # 认证机制
│   │   └ overview.md
│   │
│   ├── sniffing/              # 协议嗅探
│   │   ├── enable.md          # 嗅探开关
│   │   ├── sniffing-types.md  # 嗅探协议类型
│   │   ├── ports.md           # 嗅探端口范围
│   │   └ overview.md
│   │
│   ├── script/                # 脚本功能
│   │   ├── shortcuts.md       # 快捷脚本
│   │   ├── rule-script.md     # 规则脚本
│   │   └ overview.md
│   │
│   ├── listeners/             # 入站监听（源码: adapter/inbound/*.go）
│   │   ├── mixed.md           # Mixed Port
│   │   ├── socks.md           # SOCKS 入站
│   │   ├── http.md            # HTTP 入站
│   │   ├── tun.md             # TUN 入站
│   │   ├── redir.md           # Redirect 入站
│   │   ├── tproxy.md          # TProxy 入站
│   │   └ overview.md
│   │
│   ├── provider/              # Provider（源码: adapter/provider/*.go）
│   │   ├── proxy-provider.md  # Proxy Provider
│   │   ├── rule-provider.md   # Rule Provider
│   │   ├── health-check.md    # 健康检查
│   │   ├── override.md        # 覆盖配置
│   │   └ overview.md
│   │
│   ├── ntp/                   # NTP 同步（源码: ntp/*.go）
│   │   ├── enable.md          # NTP 开关
│   │   ├── server.md          # NTP 服务器
│   │   └ overview.md
│   │
│   ├── experimental/          # 实验性功能
│   │   ├── quic-go.md         # QUIC 实现
│   │   ├── udp-over-tcp.md    # UDP over TCP
│   │   └ overview.md
│   │
│   └── index.md               # mihomo 总索引
│
├── crypto/
│   ├── aead-basics.md         # AEAD 原理
│   ├── key-exchange.md        # 密钥交换原理（X25519）
│   ├── hkdf-theory.md         # HKDF 理论
│   ├── tls-crypto.md          # TLS 加密
│   └ overview.md
│
├── protocol/
│   ├── tls-handshake.md       # TLS 握手流程
│   ├── socks5-spec.md         # SOCKS5 RFC 1928
│   ├── http-proxy-spec.md     # HTTP 代理规范
│   ├── tcp-basics.md          # TCP 基础
│   ├── udp-basics.md          # UDP 基础
│   ├── quic-basics.md         # QUIC 基础（用于 Hysteria2/TUIC）
│   └ overview.md
│
├── network/
│   ├── happy-eyeballs.md      # RFC 6555 Happy Eyeballs
│   ├── connection-pool.md     # 连接池原理
│   ├── dns-resolution.md      # DNS 解析原理
│   ├── gfw.md                 # GFW 原理
│   ├── proxy-detection.md     # 代理检测原理
│   └ overview.md
│
├── programming/
│   ├── boost-asio.md          # Boost.Asio 协程
│   ├── cpp23-coroutine.md     # C++23 协程
│   ├── constexpr.md           # 编译期计算
│   ├── pmr-concepts.md        # PMR 概念
│   ├── error-handling.md      # 错误处理模式
│   ├── go-concurrency.md      # Go 并发模式（mihomo相关）
│   └ overview.md
│
├── glossary.md                # 术语表
└── overview.md
```

---

## 三、Wiki-sync Skill 详细设计

### Skill 文件结构

```
H:/PrismWiki/skills/prism-wiki-sync/
├── SKILL.md                   # Skill 定义（触发条件、描述）
├── checks.md                  # 检查项详细说明
├── report-template.md         # 报告模板
├── references/
│   ├── prism-structure.md     # Prism 源码结构参考（用于检查覆盖度）
│   ├── mihomo-structure.md    # Mihomo 源码结构参考
│   └ wiki-structure.md        # Wiki 目录结构参考
```

### SKILL.md 内容

```markdown
---
name: prism-wiki-sync
description: 检查知识库与源码同步状态。Push时全面检查，Release时深度检查。
---

触发条件:
- Push 到远程仓库
- Release (tag v*)
- 手动 `/wiki-sync`

检查模式:
- normal: Push时的全面检查
- deep: Release时的深度检查

输出:
- 严重问题 → 终端输出
- 完整报告 → .claude/wiki-sync-report.md

源码位置:
- Prism: I:/code/Prism/
- Mihomo: I:/code/Prism/logs/mihomo-Meta/

详细检查项见 checks.md
```

### checks.md 内容概要

```markdown
# Wiki-sync 检查项

## 普通检查（Push）

### 1. 源码路径验证
检查 wiki 中标注的源码路径是否存在
- 格式: `源码: include/prism/xxx.hpp:123`
- 检查: 文件存在 + 行号有效

### 2. 函数签名变化
检查 wiki 中描述的函数签名是否与源码一致
- 对比: 函数名、参数类型、返回类型
- 标注: `签名已变化: func() -> void → func(int) -> error`

### 3. 调用链变化
检查 wiki 中记录的调用关系是否仍然有效
- 调用者是否还调用该函数
- 被调用者是否还存在

### 4. 枚举/常量变化
检查 wiki 中列出的枚举值是否与源码一致
- 新增成员
- 删除成员
- 值变化

### 5. 类结构变化
检查类成员是否变化
- 新增/删除成员变量
- 新增/删除成员函数
- 继承关系变化

### 6. 配置项变化
检查 configuration.json 是否变化
- 字段名变化
- 默认值变化
- 新增/删除字段

### 7. 错误码变化
检查 fault::code 枚举是否变化

### 8. 协议兼容性
检查协议格式是否有变化影响客户端对接

### 9. 测试覆盖
检查新增模块是否有对应测试

### 10. 依赖关系
检查模块间依赖是否变化

### 11. 文档结构
检查新增源码是否有对应 wiki 页面

## 深度检查（Release）

### 1. 文档一致性
同一模块多个页面是否矛盾

### 2. 链接完整性
所有 wikilink 目标是否存在

### 3. 覆盖度检查
源码目录 vs wiki 目录覆盖度

### 4. 变更影响分析
函数改名后所有引用是否更新

### 5. 新功能检测
新增源码文件是否需要新建 wiki
```

---

## 四、实现任务规划

### 执行约束
- **并行上限**: 最多 3 个 agent 同时运行
- **依赖考虑**: 目录结构创建完成后才能写内容；链接引用的目标页面必须先存在
- **任务粒度**: 每个任务要足够详细，压缩上下文后仍可执行

### 任务分组（考虑依赖关系）

#### 第一阶段：基础结构（可并行）

**任务 1.1: 创建目录结构**
- 创建 core/, dev/, docs/, ref/, skills/ 的完整目录树
- 依赖: 无
- 输出: 空目录结构

**任务 1.2: 重写 SCHEMA.md**
- 定义新的目录结构规范
- 定义各层详细程度标准
- 定义调用链标记格式
- 定义 wikilink 规范
- 依赖: 无

**任务 1.3: 重写 index.md**
- 总索引，指向各层 overview
- 更新为新的目录结构
- 依赖: 无

#### 第二阶段：核心页面（依赖第一阶段）

**任务 2.1: core/overview + architecture + startup + flow (1 agent)**
- core/overview.md — core 层总览
- core/architecture.md — 六阶段流水线架构（基于 CLAUDE.md §架构概览）
- core/startup.md — 启动流程 9 步（基于 CLAUDE.md §启动流程）
- core/flow.md — 协议处理流程（基于 CLAUDE.md §协议处理流程）
- 源码参考: CLAUDE.md, main.cpp

**任务 2.2: dev/overview + coding/ + building/ (1 agent)**
- dev/overview.md — dev 层总览
- dev/coding/*.md — 编码规范（基于 CLAUDE.md §命名与编码规范）
- dev/building/*.md — 构建系统（基于 CLAUDE.md §构建系统结构）
- 源码参考: CLAUDE.md

**任务 2.3: docs/overview + all pages (1 agent)**
- docs/*.md — 所有使用指南页面
- 源码参考: docs/tutorial/, configuration.json

#### 第三阶段：core 模块页面（依赖第二阶段 overview 存在）

**任务 3.1: core/agent/ + core/channel/ (1 agent)**
- agent: front/, session/, worker/, account/, dispatch/, config, context
- channel: transport/, connection/, eyeball/, health, adapter
- 源码参考: include/prism/agent/, include/prism/channel/

**任务 3.2: core/pipeline/ + core/protocol/ (1 agent)**
- pipeline: primitives, protocols/
- protocol: common/, http/, socks5/, trojan/, vless/, shadowsocks/, tls/, analysis
- 源码参考: include/prism/pipeline/, include/prism/protocol/

**任务 3.3: core/stealth/ (1 agent)**
- stealth: scheme, executor, registry, native, reality/, shadowtls/, restls/, anytls/, trusttunnel/, ech/
- 源码参考: include/prism/stealth/, src/prism/stealth/

#### 第四阶段：core 模块页面续

**任务 4.1: core/recognition/ + core/resolve/ (1 agent)**
- recognition: recognition, result, confidence, layered_pipeline, probe/
- resolve: router, dns/
- 源码参考: include/prism/recognition/, include/prism/resolve/

**任务 4.2: core/multiplex/ + core/crypto/ + core/outbound/ (1 agent)**
- multiplex: core, bootstrap, duct, parcel, config, smux/, yamux/
- crypto: aead, hkdf, x25519, blake3, block, base64
- outbound: proxy, direct
- 源码参考: include/prism/multiplex/, include/prism/crypto/, include/prism/outbound/

**任务 4.3: core/infrastructure pages (1 agent)**
- memory/, fault/, exception/, trace/, transformer/, loader/
- infrastructure.md
- 源码参考: include/prism/memory/, fault/, exception/, trace/, transformer/, loader/

#### 第五阶段：dev 详细页面

**任务 5.1: dev/testing/ + dev/debugging/ (1 agent)**
- testing: framework, writing, commands, benchmark, stress, concurrency
- debugging: connection, protocol, memory, performance, tls, log-analysis, common-issues
- 源码参考: CLAUDE.md, tests/, .claude/skills/

**任务 5.2: dev/performance/ + dev/extending/ + dev/bugs/ (1 agent)**
- performance: tuning, profiling, report
- extending: protocol, stealth, module
- bugs: template
- 源码参考: docs/prism/performance-report.md, .claude/skills/protocol-handler

**任务 5.3: dev/roadmap (1 agent)**
- roadmap.md — 开发路线图
- 源码参考: CLAUDE.md §开发路线图, plan.md

#### 第六阶段：ref 基础 + mihomo

**任务 6.1: ref/overview + ref/crypto/ + ref/protocol/ + ref/network/ (1 agent)**
- ref/overview.md
- ref/crypto/ — 加密知识
- ref/protocol/ — 协议知识
- ref/network/ — 网络知识
- 源码参考: 知识积累，不依赖特定源码

**任务 6.2: ref/mihomo/protocols/ + ref/mihomo/transport/ (1 agent)**
- protocols: 所有出站协议（socks5, trojan, vless, vmess, ss, hysteria2, tuic, wireguard 等）
- transport: shadowtls, restls, v2ray-plugin 等
- 源码参考: I:/code/Prism/logs/mihomo-Meta/adapter/outbound/, transport/

**任务 6.3: ref/mihomo/其他模块 (1 agent)**
- mux/, proxy-groups/, rules/, dns/, tun/, config/, compatibility/, implementation/
- sniffing/, listeners/, provider/, ntp/, experimental/
- 源码参考: I:/code/Prism/logs/mihomo-Meta/ 各目录

#### 第七阶段：ref/programming + glossary

**任务 7.1: ref/programming/ + glossary (1 agent)**
- programming: boost-asio, cpp23-coroutine, constexpr, pmr-concepts, error-handling, go-concurrency
- glossary.md — 术语表
- 源码参考: CLAUDE.md, 知识积累

#### 第八阶段：Skill 实现

**任务 8.1: skills/prism-wiki-sync/ 全部文件 (1 agent)**
- SKILL.md
- checks.md
- report-template.md
- references/prism-structure.md
- references/mihomo-structure.md
- references/wiki-structure.md
- 依赖: 所有 wiki 页面已创建

#### 第九阶段：清理旧文件

**任务 9.1: 删除旧目录结构**
- 删除 incoming/, recognition/recognition/, 等旧目录
- 删除 docs/ 下的旧文件（agent.md, architecture.md 等）
- 保留 SCHEMA.md, index.md（已重写）

---

## 五、每个任务的详细内容

### 任务 1.1 详细：创建目录结构

```bash
# 要创建的目录
mkdir -p core/agent/{front,session,worker,account,dispatch}
mkdir -p core/channel/{transport,connection,eyeball,adapter}
mkdir -p core/pipeline/protocols
mkdir -p core/protocol/{common,http,socks5,trojan,vless,shadowsocks,tls}
mkdir -p core/stealth/{reality,shadowtls,restls,anytls,trusttunnel,ech}
mkdir -p core/recognition/probe
mkdir -p core/resolve/dns/detail
mkdir -p core/multiplex/{smux,yamux}
mkdir -p core/crypto
mkdir -p core/outbound
mkdir -p core/memory
mkdir -p core/fault
mkdir -p core/exception
mkdir -p core/trace
mkdir -p core/transformer
mkdir -p core/loader

mkdir -p dev/coding
mkdir -p dev/testing
mkdir -p dev/debugging
mkdir -p dev/building
mkdir -p dev/performance
mkdir -p dev/extending
mkdir -p dev/bugs

mkdir -p docs

mkdir -p ref/mihomo/{protocols,transport,mux,proxy-groups,rules,dns,tun,config,compatibility,implementation,sniffing,script,listeners,provider,ntp,experimental}
mkdir -p ref/crypto
mkdir -p ref/protocol
mkdir -p ref/network
mkdir -p ref/programming

mkdir -p skills/prism-wiki-sync/references
```

### 任务 2.1 详细：core/ 架构页面

**core/architecture.md 内容要点**:
- 六阶段流水线图（ASCII）
```
入站 → 识别 → 分发 → 道 → 通道 → 出站
```
- 每阶段职责说明
- 各阶段对应模块
- wikilink 到各模块 overview

**core/startup.md 内容要点**:
- main.cpp 启动顺序 9 步
- 每步调用的函数（wikilink）
- 配置文件查找顺序
- 注意事项

**core/flow.md 内容要点**:
- 协议处理流程（ASCII 流程图）
- 从 listener 接收到 tunnel 转发
- Recognition 三阶段流水线
- wikilink 到各模块详细页面

### 任务 2.2 详细：dev/coding/ 页面

**每个页面内容来源**:
- naming.md — CLAUDE.md §命名与编码规范
- coroutine.md — CLAUDE.md §协程约定、§协程纯度要求（禁止表）
- pmr.md — CLAUDE.md §PMR内存策略
- doxygen.md — CLAUDE.md §命名与编码规范-注释
- lifecycle.md — CLAUDE.md §生命周期安全
- error.md — CLAUDE.md §错误处理（双轨策略）

### 任务 3.1 详细：core/agent/ 页面

**每个页面要包含**:
- 模块职责说明
- 文件结构（头文件/源文件）
- 主要类/函数详解
- 调用链（调用者、被调用者）
- 与其他模块交互

**listener.md 内容要点**:
- 源码: include/prism/agent/front/listener.hpp
- 功能: 监听端口，接受连接，计算亲和性哈希
- 调用: 调用 [[agent/front/balancer|负载均衡器]] 的 select()
- 被调用: main.cpp 启动

**balancer.md 内容要点**:
- 源码: include/prism/agent/front/balancer.hpp
- 功能: 选择 worker 分发连接
- 调用: [[agent/worker/worker|Worker]] 的 post()
- 被调用: [[agent/front/listener|监听器]]

**session.md 内容要点**:
- 源码: include/prism/agent/session/session.hpp
- 功能: 会话生命周期管理
- 调用: [[recognition/recognition|协议识别]], [[pipeline/protocols|协议处理器]]
- 被调用: [[agent/worker/launch|会话启动]]

（其他页面类似，每个都要写详细）

### 任务 6.2 详细：ref/mihomo/protocols/

**每个协议页面内容**:
- 协议概述
- mihomo 实现位置（源码文件）
- 配置格式（YAML 示例）
- 与 Prism 的兼容性
- 实现要点（关键代码逻辑）
- wikilink 到相关传输层

**vless.md 内容要点**:
- 源码: adapter/outbound/vless.go, transport/vless/
- 配置格式示例
- 支持: TCP, XUDP, mux
- Reality 支持
- 与 Prism vless 对比

**hysteria2.md 内容要点**:
- 源码: adapter/outbound/hysteria2.go
- 基于 QUIC
- 配置格式
- 拥塞控制（bbr, brutal）
- Prism 不支持（记录兼容性）

（其他协议页面类似）

### 任务 8.1 详细：skills/prism-wiki-sync/

**SKILL.md**:
- 触发条件定义
- 检查模式定义
- 输出格式定义
- 源码位置定义

**checks.md**:
- 所有检查项详细说明
- 每个检查项的具体实现逻辑
- 错误类型分类

**report-template.md**:
```markdown
# Wiki-sync 检查报告

检查时间: {{timestamp}}
检查模式: {{mode}} (normal/deep)

## 严重问题

{{critical_issues}}

## 一般问题

{{normal_issues}}

## 覆盖度统计

- core/ 覆盖度: {{core_coverage}}%
- dev/ 覆盖度: {{dev_coverage}}%
- ref/ 覆盖度: {{ref_coverage}}%

## 建议

{{suggestions}}
```

**references/prism-structure.md**:
- Prism 源码目录结构
- 每个模块的头文件/源文件对应
- 用于检查覆盖度

**references/wiki-structure.md**:
- Wiki 目录结构
- 每个页面对应的源码文件
- 用于检查覆盖度

---

## 六、任务依赖图

```
阶段1 (并行3):
  [1.1 创建目录] [1.2 SCHEMA.md] [1.3 index.md]
         ↓
阶段2 (并行3):
  [2.1 core架构页面] [2.2 dev基础页面] [2.3 docs页面]
         ↓
阶段3 (并行3):
  [3.1 agent+channel] [3.2 pipeline+protocol] [3.3 stealth]
         ↓
阶段4 (并行3):
  [4.1 recognition+resolve] [4.2 multiplex+crypto+outbound] [4.3 infrastructure]
         ↓
阶段5 (并行3):
  [5.1 testing+debugging] [5.2 performance+extending+bugs] [5.3 roadmap]
         ↓
阶段6 (并行3):
  [6.1 ref基础] [6.2 mihomo协议+传输] [6.3 mihomo其他]
         ↓
阶段7 (并行1):
  [7.1 programming+glossary]
         ↓
阶段8 (并行1):
  [8.1 skill实现]
         ↓
阶段9 (并行1):
  [9.1 清理旧文件]
```

---

## 七、验收标准

完成后应满足:
1. 所有目录结构完整创建
2. 所有页面内容基于实际源码
3. wikilink 正确指向目标页面
4. SCHEMA.md 定义了完整规范
5. index.md 是完整索引
6. skill 可正常运行检查
7. 无重复知识内容（同一知识点只在一处详细写）
8. 清理了所有旧目录/文件