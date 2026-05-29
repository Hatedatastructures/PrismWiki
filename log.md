# Wiki Log

> Chronological record of all wiki actions. Append-only.
> Format: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> When this file exceeds 500 entries, rotate: rename to log-YYYY.md, start fresh.

## [2026-05-17] refactor | PrismWiki 知识库四层分离架构重构

### 背景
- 知识库结构混乱，架构不清晰
- 文档不够详细，缺少调用链标注
- mihomo 参考资料不完整
- 需要 wiki-sync skill 检查同步状态

### 架构设计
- 四层分离：core/（详细）、dev/（开发）、docs/（用户）、ref/（参考）
- 知识不重复原则：同一知识点只在一处详细写，其他用 wikilink 引用
- core/ 层复杂函数逐行解释，标注源码位置和调用链

### 执行过程（19 Phases, 44 Tasks）

#### Phase 1-3: 目录结构 + 基础文件
- 创建 core/、dev/、docs/、ref/、skills/ 目录结构
- 重写 SCHEMA.md 定义四层架构
- 重写 index.md 四层索引
- 创建 core/overview.md、architecture.md、startup.md、flow.md、infrastructure.md

#### Phase 4-5: dev/ + docs/ 层
- dev/overview.md + coding/（naming、coroutine、pmr、doxygen、lifecycle、error）
- dev/building/（cmake、dependencies、commands、options）
- docs/ 所有用户指南页面（getting-started、deployment 等）

#### Phase 6-12: core/ 模块
- agent/（overview、config、context、front、session、worker、account、dispatch）
- channel/（overview、health、transport、connection、eyeball、adapter）
- pipeline/ + protocol/（overview、primitives、protocols、common、http、socks5、trojan、vless、shadowsocks、tls）
- stealth/（overview、scheme、executor、registry、native、reality、shadowtls、restls、anytls、trusttunnel、ech）
- recognition/ + resolve/（overview、recognition、result、confidence、probe、dns）
- multiplex/（overview、bootstrap、duct、parcel、smux、yamux）
- crypto/ + outbound/（aead、hkdf、x25519、blake3、block、base64、proxy、direct）
- 基础设施（memory、fault、exception、trace、transformer、loader）

#### Phase 13-15: dev/ 其他 + ref/mihomo/
- dev/testing/、debugging/、performance/、extending/、bugs/、roadmap
- ref/overview + crypto/protocol/network
- mihomo/protocols/（22 个协议）
- mihomo/transport/、mux/、proxy-groups/、rules/、dns/、tun/
- mihomo/config/、compatibility/、implementation/
- mihomo/sniffing/、listeners/、provider/、ntp/、experimental/、script/

#### Phase 16-17: ref/programming/ + wiki-sync skill
- ref/programming/（overview、boost-asio、cpp23-coroutine、constexpr、pmr-concepts、error-handling、go-concurrency）
- ref/glossary.md（术语表）
- skills/prism-wiki-sync/（SKILL.md、checks.md、report-template.md、source-mapping.md）

#### Phase 18: 清理旧目录
删除：incoming/、agent/、channel/、crypto/、multiplex/、pipeline/、protocol/、stealth/、resolve/、memory/、fault/、exception/、trace/、transformer/、loader/、outbound/、client/、dispatch/、infrastructure/、performance/、recognition/、bugs/、.plan/
删除：exception.md、fault.md、loader.md、memory.md、outbound.md、trace.md、transformer.md

### 最终目录结构
```
wiki/
├── core/              # 模块实现细节（详细）
├── dev/               # 开发规范、排障方法（详细）
├── docs/              # 使用指南（简单）
├── ref/               # 参考知识 + mihomo（中等）
├── skills/            # prism-wiki-sync skill
├── SCHEMA.md
├── index.md
├── log.md
└── README.md
```

### 页面统计
- core/: ~120 页面
- dev/: ~45 页面
- docs/: ~12 页面
- ref/: ~180 页面（含 mihomo/ ~100 页面）
- skills/: 4 页面
## [2026-05-17] verify | 深度验证与修复

### 验证结果
- 三个 Explore agents 完成全面检查
- 发现问题：32 个文件缺 frontmatter、144 个缺 title、source 路径不一致、5 个协议缺字段

### 修复内容

#### Phase 1: Frontmatter 补充
- ref/mihomo/protocols/*.md：24 个协议文件添加 frontmatter
- docs/*.md：10 个文件补充完整 frontmatter
- index.md、SCHEMA.md：添加 frontmatter
- core/infrastructure.md、core/overview.md：补充 source 字段

#### Phase 2: Source 路径统一
- 10 个文件：相对路径 `include/prism/...` → 绝对路径 `include/prism/...`

#### Phase 3: Mihomo 协议字段补充
- VLESS：5 个新字段（packet-addr、packet-encoding、encryption、ws-headers、servername）
- VMess：6 个新字段（servername、packet-addr、packet-encoding、global-padding、authenticated-length、client-fingerprint）
- Trojan：3 个新字段（ech-opts、reality-opts、client-fingerprint）
- Hysteria2：5 个新字段（udp-mtu、4 个 QUIC 窗口参数）
- TUIC：6 个新字段（ip、disable-sni、recv-window-conn、recv-window、disable-mtu-discovery、max-datagram-frame-size）

#### Phase 4: 空文件填充
- ref/logging/spdlog.md：添加 spdlog 文档
- ref/programming/cpp-23.md：添加 C++23 新特性文档
- ref/protocol/http-1-1.md：添加 HTTP/1.1 文档
- ref/mihomo/protocols/overview.md：添加 frontmatter

### 最终统计
- 总页面：389
- 有 frontmatter：385（99%）
- source 路径统一：46 个绝对路径，0 个相对路径
- Mihomo 协议：25 个，字段完整
- 四层分离架构完整实现
- 所有 Prism 模块详细文档
- mihomo 22+ 协议完整参考
- wiki-sync skill 用于同步检查
- 知识库规范（SCHEMA.md）确立
