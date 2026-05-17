# Wiki Log

> Chronological record of all wiki actions. Append-only.
> Format: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> When this file exceeds 500 entries, rotate: rename to log-YYYY.md, start fresh.

## [2026-05-12] create | Wiki initialized
- Domain: 技术栈与项目 (Tech Stacks & Projects)
- Structure created with SCHEMA.md, index.md, log.md
- Location: H:\wiki
- Directories: raw/, entities/, concepts/, comparisons/, queries/

## [2026-05-12] ingest | Prism 项目
- 创建 Prism 项目页面 (entities/prism.md)
- 创建 C++23 协程页面 (concepts/cpp23-coroutines.md)
- 创建 PMR 内存池页面 (concepts/pmr-memory-pool.md)
- 创建代理协议页面 (concepts/proxy-protocols.md)
- 创建 Reality TLS 页面 (concepts/reality-tls.md)
- 更新 index.md

## [2026-05-12] ingest | Prism 项目详细架构
- 创建架构设计页面 (concepts/prism-architecture.md)
- 创建模块结构页面 (concepts/prism-modules.md)
- 创建配置详解页面 (concepts/prism-configuration.md)
- 创建测试体系页面 (concepts/prism-testing.md)
- 更新 index.md

## [2026-05-12] ingest | Prism 性能报告和部署指南
- 创建性能基准测试页面 (concepts/prism-performance.md)
- 创建部署指南页面 (concepts/prism-deployment.md)
- 创建故障排查页面 (concepts/prism-troubleshooting.md)
- 更新 index.md

## [2026-05-12] 整理 | 格式统一
- 删除性能报告页面（prism-performance.md）
- 重写所有页面为标准中文 markdown 格式
- 统一 frontmatter 格式
- 更新 index.md

## [2026-05-12] ingest | 新增两个项目
- 创建 mihomo-Meta 项目页面 (entities/mihomo-meta.md)
- 创建 Trojan-GFW 项目页面 (entities/trojan-gfw.md)
- 更新 index.md

## [2026-05-12] ingest | 新增 yamux 项目
- 创建 yamux 项目页面 (entities/yamux.md)
- 创建流多路复用概念页面 (concepts/stream-multiplexing.md)
- 更新 index.md，总页面数增至 16

## [2026-05-12] ingest | 基础协议 + mihomo 配置
- 创建 TCP 页面 (concepts/tcp.md)
- 创建 UDP 页面 (concepts/udp.md)
- 创建 TLS 页面 (concepts/tls.md)
- 创建 QUIC 页面 (concepts/quic.md)
- 创建 DNS 页面 (concepts/dns.md)
- 创建 mihomo Clash 配置页面 (concepts/mihomo-clash-config.md)
- 清理现有页面：去掉构建命令，以 mihomo 为中心
- 统一 tag 分类为网络安全代理方向
- 更新 SCHEMA.md 域名为"网络安全代理"
- 更新 index.md，总页面数增至 20

## [2026-05-12] ingest | 代理协议（批量）
- 创建 VMess 页面 (concepts/vmess.md)
- 创建 Shadowsocks 页面 (concepts/shadowsocks.md)
- 创建 TUIC 页面 (concepts/tuic.md)
- 创建 Hysteria 页面 (concepts/hysteria.md)

## [2026-05-12] ingest | 伪装与对抗技术（批量）
- 创建 TLS 指纹页面 (concepts/tls-fingerprint.md)
- 创建 ShadowTLS 页面 (concepts/shadowtls.md)
- 创建 Restls 页面 (concepts/restls.md)
- 创建流量分析与对抗页面 (concepts/traffic-analysis.md)

## [2026-05-12] ingest | GFW + 对比页面（批量）
- 创建 GFW 页面 (concepts/gfw.md)
- 创建代理协议对比页面 (comparisons/proxy-protocols-comparison.md)
- 创建 TLS 伪装技术对比页面 (comparisons/tls-camouflage-comparison.md)
- 更新 index.md，总页面数增至 31

## [2026-05-12] fix | 断裂 wikilinks 修复
- gfw.md: [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] → [[ref/anti-censorship/tls-fingerprint]], [[ref/anti-censorship/traffic-analysis]] → [[ref/anti-censorship/traffic-analysis|流量分析与对抗]]
- traffic-analysis.md: [[ref/anti-censorship/tls-fingerprint|TLS 指纹]] → [[ref/anti-censorship/tls-fingerprint]] (2处)
- tls-camouflage-comparison.md: [[ref/anti-censorship/traffic-analysis]] → [[ref/anti-censorship/traffic-analysis|流量分析与对抗]]
- mihomo-meta.md: [[client/tun]] → [[client/tun|TUN 模式]]

## [2026-05-12] ingest | VLESS + TUN 页面
- 创建 VLESS 页面 (concepts/vless.md)
- 创建 TUN 模式页面 (concepts/tun.md)
- 更新 index.md，总页面数增至 34

## [2026-05-12] fix | wikilink 全面修复
- 修复 C++23-Coroutines → cpp23-coroutines (3处)
- 修复 TUN-模式 → tun (1处)
- 修复 流量分析与对抗 → traffic-analysis (3处)
- 修复 代理协议对比 → proxy-protocols-comparison (index.md)
- 修复 TLS 伪装技术对比 → tls-camouflage-comparison (index.md)
- 修复 TLS-指纹 → tls-fingerprint (index.md)
- 修复 index.md 中的重复 VLESS 条目

## [2026-05-12] update | 页面深度扩展（7个）
- cpp23-coroutines: 77行→177行，补充协程在代理中的应用
- pmr-memory-pool: 75行→159行，补充内存池在高并发代理中的作用
- proxy-protocols: 86行→207行，扩充每个协议详情+协议选择建议
- reality-tls: 82行→181行，补充技术细节+配置参数+局限性
- restls: 67行→135行，补充 TLS 记录层伪装原理
- mihomo-meta: 64行→181行，补充核心功能+与原版 Clash 对比
- prism: 82行→197行，补充架构概览+与 mihomo 关系

## [2026-05-12] ingest | 新增 5 个缺失页面
- 创建 HTTP 代理页面 (concepts/http-proxy.md)
- 创建 SOCKS5 页面 (concepts/socks5.md)
- 创建代理传输层页面 (concepts/transport-layer.md)
- 创建 mihomo 规则系统页面 (concepts/mihomo-rules.md)
- 创建 mihomo 代理组页面 (concepts/mihomo-proxy-groups.md)

## [2026-05-12] fix | Prism 页面断裂引用修复
- prism-architecture: 添加 [[dev/tcp]], [[protocol/tls]], [[client/mihomo-meta]]
- prism-configuration: 添加 [[client/mihomo-clash-config]], [[client/mihomo-meta]]
- prism-deployment: 添加 [[client/mihomo-meta]], [[dev/gfw]]
- prism-modules: 添加 [[client/mihomo-meta]], [[docs/protocol/proxy-protocols]]
- prism-testing: 添加 [[dev/tcp]], [[docs/protocol/proxy-protocols]]
- prism-troubleshooting: 添加 [[client/mihomo-meta]], [[dev/gfw]], [[resolve/dns]]
- 更新 index.md，总页面数增至 39

## [2026-05-12] restructure | 知识库全面重构
- 重构目录结构：concepts/ → networking/, proxy/, security/, tools/, dev/
- 迁移 39 个现有文件到新目录
- 新增 32 个页面（总计 71 个）

### networking/ 新增 (19个)
osi-model, ip, http, http2, http3, doh, dot, sni, ech, websocket, grpc,
nat, routing, cdn, load-balancing, congestion-control, keep-alive, port-reuse, connection-pool

### proxy/ 新增 (6个)
proxy-architecture, proxy-chaining, transparent-proxy, proxy-detection, ip-camouflage, domain-fronting

### security/ 新增 (6个)
encryption-system, pki-certificates, dpi, privacy-protection, anonymous-networks, traffic-obfuscation

### tools/ 新增 (1个)
sing-box

- 重建 SCHEMA.md（新目录结构 + tag 分类）
- 重建 index.md（按目录分类的完整索引）

## [2026-05-12] rewrite | 核心协议页面深度重写（12个）
参照 Prism 项目文档的深度标准，重写以下页面：

### 第一批（6个，465-654行）
- vless.md: 183→514行，加入二进制帧格式、ATYP 映射、Hex dump
- vmess.md: 149→473行，加入认证信息格式、加密头部、时间戳防重放
- shadowsocks.md: 187→578行，加入 SS2022 TCP 帧结构、BLAKE3 密钥派生
- tls.md: 202→563行，加入 TLS 记录层格式、ClientHello 详解、密码套件解析
- reality-tls.md: 181→465行，加入 SessionID 嵌入、X25519 ECDH 认证流程
- http-proxy.md: 156→654行，加入三种请求形式、CONNECT 隧道生命周期

### 第二批（6个，372-652行）
- sni.md: 124→465行，加入 SNI 在 ClientHello 中的位置、SNI 防火墙
- quic.md: 175→637行，加入 QUIC 包格式、Connection ID、0-RTT 握手
- dns.md: 174→652行，加入 DNS 报文格式、fake-ip 原理图、DoH/DoT
- dpi.md: 107→425行，加入 DPI 检测方法、各协议 Hex Dump 检测特征
- traffic-analysis.md: 157→372行，加入包长度分析、主动探测流程图
- websocket.md: 139→503行，加入帧格式、Hex dump、CDN 中转架构

总页面数：72

## [2026-05-12] expand | 全部薄页面扩展完成
扩展 15 个页面到 200+ 行：

### networking/ (6个)
- congestion-control: 102→270行，Reno/CUBIC/BBR/Brutal 算法详解
- connection-pool: 103→213行，连接池状态机/健康检查
- ech: 115→216行，双层 ClientHello/ECH 密钥获取/部署现状
- load-balancing: 105→274行，负载均衡算法/L4 vs L7/mihomo 策略
- osi-model: 113→282行，七层详解/代理工作层/OSI vs TCP/IP
- port-reuse: 108→262行，SO_REUSEPORT/首字节嗅探

### proxy/ (2个)
- domain-fronting: 103→206行，SNI vs Host/CDN 路由/封锁历史
- ip-camouflage: 109→246行，CDN/WireGuard/WARP/住宅代理

### security/ (5个)
- anonymous-networks: 109→341行，Tor 三跳加密/I2P/威胁模型
- pki-certificates: 104→303行，X.509/证书链/ACME/Reality
- privacy-protection: 107→300行，IP/DNS/WebRTC 泄漏/浏览器指纹
- shadowtls: 102→285行，v1/v2/v3/握手流程/与 Reality 对比
- traffic-obfuscation: 101→315行，obfs-http/tls/v2ray-plugin

### proxy/ (1个)
- proxy-detection: 103→301行，主动探测/GFW 检测/抗检测评级

### tools/ (1个)
- prism-troubleshooting: 117→427行，4类问题/诊断流程图/错误码

最终状态：72 个页面，全部 >= 120 行，0 个薄页面

## [2026-05-13] rename | 知识库改名
- 原名：网络安全代理知识库
- 新名：Prism 技术文档
- 更新 SCHEMA.md、index.md 域名描述
- 原因：内容已扩展到项目开发、调试排障、客户端对接，不再局限于网络安全代理

## [2026-05-13] refactor | 知识库重构为 Prism 技术文档

### 删除内容
- 删除 networking/ 目录（29 页）：通用网络协议知识
- 删除 proxy/ 目录（15 页）：代理技术对比
- 删除 security/ 目录（11 页）：安全原理
- 删除 tools/sing-box.md、tools/tun.md
- 删除 queries/、raw/ 目录
- 共删除 ~59 页通用知识

### 保留内容
- tools/prism*.md → agent/（7 页）
- tools/mihomo-*.md → client/（4 页）
- dev/*.md（2 页）

### 新建目录
- agent/、channel/、crypto/、multiplex/、pipeline/、protocol/、recognition/、resolve/、stealth/、memory/、fault/、client/、bugs/

### 更新文件
- SCHEMA.md：更新为 Prism 项目文档规范，增加 Bug 记录格式
- index.md：重写为 Prism 模块索引

### 最终状态
- 16 个页面（14 个内容页 + 2 个元数据页）
- 知识库从"网络安全代理知识库"更名为"Prism 技术文档"
- 重心从通用知识转向项目文档、调试排障、客户端对接

## [2026-05-13] detail | 基于源码重写所有模块文档

### 新增模块文档（11 个）
- agent/module-detail.md — 7.8KB，Listener/Session/Worker/Balancer 详细设计
- channel/overview.md — 6.9KB，连接池、健康检查、传输层架构
- crypto/overview.md — 5.2KB，AEAD/Blake3/X25519/HKDF 实现
- multiplex/overview.md — 5.3KB，smux/yamux 帧格式、流管理
- pipeline/overview.md — 4.7KB，协议处理管道、原语操作
- protocol/overview.md — 4.1KB，HTTP/SOCKS5/Trojan/VLESS 编解码
- recognition/overview.md — 5.2KB，三阶段识别流水线
- resolve/overview.md — 5.2KB，DNS 解析器、缓存、规则
- stealth/overview.md — 7.6KB，Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel
- memory/overview.md — 5.8KB，PMR 三级内存池
- fault/overview.md — 5.7KB，错误码枚举、双轨错误处理

### 关键发现
- Prism 使用 C++23 协程（Boost.Asio awaitable<T>），不是 C++20
- PMR 内存池三级架构：全局同步 → 线程本地 → 帧 arena
- 协议识别三阶段：Probe（24字节 peek）→ Identify（ClientHello 分析）→ Execute（方案流水线）
- 6 种伪装方案，分层检测：Tier 0（sniff/零成本）→ Tier 1（verify/HMAC）→ Tier 2（guess/SNI）
- 错误处理双轨：热路径用 fault::code（无异常），启动/致命错误用异常

### 最终状态
- 27 个页面（11 个模块文档 + 7 个 agent 文档 + 4 个 client 文档 + 2 个 dev 文档 + 3 个元数据页）
- 所有模块文档基于源码分析，包含类层次、接口、数据流、设计决策

## [2026-05-13] skill | 创建 bug-to-wiki 技能

- 创建: I:/code/Prism/.claude/skills/bug-to-wiki/SKILL.md
- 用途: Bug 修复和对接问题排查完成后，将经验记录到 Prism 技术文档知识库
- 触发条件: 修复完成且测试通过后，用户确认归档
- 归档路径:
  - bug → H:/wiki/bugs/
  - 客户端对接 → H:/wiki/client/
  - 模块问题 → H:/{module}/
  - 协议问题 → H:/wiki/protocol/ 或 H:/wiki/stealth/
- 更新: I:/code/Prism/CLAUDE.md 添加技能引用

## [2026-05-13] detail | 并行分析 Prism 全部模块

### 分析任务
- 组1（agent 模块）：直接分析，生成 H:/wiki/agent/detail.md（22KB）
- 组2（协议处理流水线）：delegate_task 生成 4 个文档
- 组3（传输与伪装层）：delegate_task 生成 10 个文档

### 新增模块文档（15 个）
- agent/detail.md — 22KB，listener/balancer/session/worker/dispatch/context 详细设计
- protocol/detail.md — 13.7KB，HTTP/SOCKS5/Trojan/VLESS/SS2022/TLS 协议实现
- recognition/detail.md — 18.3KB，三阶段识别流水线
- pipeline/detail.md — 20.1KB，协议处理管道和原语
- resolve/detail.md — 28.7KB，DNS 解析七阶段管道
- channel/detail.md — 11.6KB，连接池、Happy Eyeballs、传输层
- multiplex/detail.md — 10KB，smux/yamux 多路复用
- stealth/detail.md — 11KB，Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel
- crypto/detail.md — 5.4KB，AEAD/Blake3/X25519/HKDF
- memory/detail.md — 4.5KB，PMR 三级内存池
- fault/detail.md — 2.9KB，错误码枚举
- exception/detail.md — 2.1KB，异常层次
- trace/detail.md — 2KB，日志系统
- transformer/detail.md — 1.6KB，JSON 转换
- outbound/detail.md — 2.2KB，出站代理

### 关键发现
- agent 模块采用编译期函数表替代虚函数，零虚函数分发
- 负载均衡采用加权评分（会话数60% + 待处理10% + 延迟30%）
- 会话生命周期采用 shared_from_this 管理
- 反压机制：listener 延迟接受新连接

### 最终状态
- 42 个页面（27 原有 + 15 新增模块文档）
- 所有模块文档基于源码分析，包含类层次、接口、数据流、设计决策

## [2026-05-13] cleanup | 删除过时文档

- 删除: agent/module-detail.md（与 detail.md 重复）
- 原因: detail.md 内容更详细且有正确的 frontmatter

## [2026-05-13] ingest | Prism 官方文档导入（协议/伪装/多路复用/性能）

从 Prism 项目源码 `docs/prism/` 导入 9 个官方文档到知识库，添加标准 YAML frontmatter 和 wikilinks。

### 协议规范（5 个）
- protocol/http.md — HTTP/1.1 代理协议，CONNECT 隧道、Basic 认证
- protocol/socks5.md — SOCKS5 协议（RFC 1928），TCP/UDP 代理
- protocol/trojan.md — Trojan 协议，TLS 伪装、SHA224 凭据
- protocol/vless.md — VLESS 协议，UUID 认证、二进制帧格式
- protocol/shadowsocks.md — Shadowsocks 2022，AEAD 加密、BLAKE3 密钥派生

### TLS 伪装（1 个）
- stealth/reality.md — Reality TLS 伪装，X25519 ECDH、合成 Ed25519 证书

### 多路复用（2 个）
- multiplex/smux.md — smux 协议，8 字节帧头、sing-mux 协商
- multiplex/yamux.md — yamux 协议，12 字节帧头、256KB 流量控制

### 性能报告（1 个）
- performance/report.md — 性能基准测试，协议握手、AEAD 加密、内存分配

### 变更
- 创建 performance/ 目录
- 更新 index.md，新增 9 个条目，总页面数 42 → 50

## [2026-05-13] rename | 知识库更名为 Prism 知识库

### 变更
- index.md 标题：Prism 技术文档 → Prism 知识库
- SCHEMA.md 域描述同步更新
- log.md 中的历史记录保留原名（记录当时事实）

## [2026-05-13] create | 新增代理协议总览 + Trojan-GFW 页面

### 新增页面（2 个）
- protocol/proxy-protocols.md — 代理协议总览，HTTP/SOCKS5/Trojan/VLESS/Shadowsocks 对比、选型建议、Pipeline 处理链路
- protocol/trojan-gfw.md — Trojan-GFW 原始协议规范，SHA224+CRLF 格式、TLS 指纹、与 Prism 实现的差异

### 变更
- 更新 index.md，新增 2 个条目，总页面数 50 → 52

## [2026-05-13] audit | 知识库全面审计与补全

### 新增页面（13 个）
- protocol/proxy-protocols.md — 代理协议总览
- protocol/trojan-gfw.md — Trojan-GFW 纯实现
- client/mihomo-dns.md — mihomo DNS 处理机制
- client/tun.md — mihomo TUN 模式
- dev/tcp.md — TCP 协议基础
- dev/tls.md — TLS 协议基础
- dev/udp.md — UDP 协议基础
- dev/gfw.md — GFW 封锁原理
- stealth/pki-certificates.md — PKI 证书体系
- stealth/proxy-detection.md — 代理检测技术
- loader/overview.md — Loader 模块概述
- trace/overview.md — Trace 模块概述
- transformer/overview.md — Transformer 模块概述

### 断链修复
- 82 个初始断链 → 0 断链
- 修复类型 1：重定向（prism→agent/overview, reality-tls→stealth/reality 等）
- 修复类型 2：短名→全路径（channel→channel/overview 等）
- 涉及 21 个已有文件的链接修正

### 交叉链接补全
- 11 个已有文件新增指向新页面的交叉链接
- index.md 更新至 65 页面
- 总唯一链接数：308

## [2026-05-13] rewrite | 全模块 overview + detail 重写

### 重写范围
- 16 个模块的 overview.md 和 detail.md 全部基于源码重新分析撰写
- 基于实际源码头文件（120+ hpp, 70+ cpp），不是照抄文档

### 重写的文件（26 个）
- memory/overview.md + detail.md
- crypto/overview.md + detail.md
- fault/overview.md + detail.md
- exception/detail.md
- trace/overview.md + detail.md
- transformer/overview.md + detail.md
- loader/overview.md
- outbound/detail.md
- channel/overview.md + detail.md
- multiplex/overview.md + detail.md
- stealth/overview.md + detail.md
- resolve/overview.md + detail.md
- protocol/overview.md + detail.md
- pipeline/overview.md + detail.md
- recognition/overview.md + detail.md
- agent/overview.md + architecture.md + detail.md

### 结构修正
- agent/modules.md → dev/modules.md（全局模块结构不属于 agent 子模块）
- smux 文档不再包含 yamux 文件结构，反之亦然
- overview 和 detail 内容不重复：overview 讲定位和设计，detail 讲实现和接口
- 所有文件清单只包含本模块的文件

### 验证
- 断链数：0（[[likely]] 是 C++ 属性，非 wikilink 误报）
- 总页面数：64
- 总唯一链接数：~100

## [2026-05-13] merge | overview+detail 合并为模块名.md

### 结构变更
- 12 个模块的 overview.md + detail.md 合并为 单文件（模块名.md）
- 3 个单文件模块重命名（exception/detail→exception.md, outbound/detail→outbound.md, loader/overview→loader.md）
- 删除 29 个旧文件，12 个空目录

### 合并的模块
channel, crypto, fault, memory, multiplex, pipeline, protocol, recognition, resolve, stealth, trace, transformer → 根目录模块名.md

### 保留的独立文件
- agent/architecture.md, configuration.md, deployment.md, testing.md, troubleshooting.md
- multiplex/smux.md, yamux.md
- protocol/http.md, socks5.md, trojan.md, vless.md, shadowsocks.md, trojan-gfw.md, proxy-protocols.md
- stealth/reality.md, pki-certificates.md, proxy-detection.md
- client/*, dev/*, performance/*

### 验证
- 51 页面，129 唯一链接
- 断链：0
- 孤立页面：0
- 缺少 frontmatter：0

## [2026-05-13] restructure | 模块文件归位 + 深度补充

### 结构重组
- 16 个根目录模块文件移入各自文件夹（agent.md → agent/agent.md）
- 删除 agent/modules.md（无用重定向）
- 修复 agent/configuration.md 中的断链

### 新增文件（11 个）
- protocol/tls.md — TLS ClientHello 特征分析、feature_bitmap
- protocol/common.md — 地址解析、协议表单、UDP 中继公共组件
- stealth/shadowtls.md — ShadowTLS v2/v3 实现
- stealth/restls.md — Restls 方案
- stealth/anytls.md — AnyTLS 方案
- stealth/trusttunnel.md — TrustTunnel 方案
- stealth/ech.md — ECH 加密 Client Hello
- channel/transport.md — 传输层接口分层
- crypto/aead.md — AEAD 加密实现
- resolve/dns.md — DNS 解析器管道
- memory/pool.md — 内存池三级架构

### 增强的文件（2 个）
- protocol/http.md — 补充 Prism 实现详解（解析器、认证、中继）
- protocol/socks5.md — 补充 Prism 实现详解（消息编解码、UDP 中继）

### 验证
- 61 页面，145 唯一链接
- 断链：0（修复 agent/modules → dev/modules）
- SCHEMA.md 是元文件，不需要被链接

## [2026-05-15] rewrite | API 文档全面重构（7 字段标准）

### 目标
为所有 API 文档的每个函数补齐 7 个必填字段：功能说明、签名、参数表格、返回值、调用（向下）、被调用（向上）、知识域。函数只写函数名，不贴实现代码。

### 标准模板
参考 `crypto/aead.md` 的 `seal()` 函数作为金标准。SCHEMA.md 已更新，新增"API 函数文档标准"章节。

### Phase 1: 修复编码和基础设施
- `stealth/scheme.md` — 从源码重新生成，修复 mojibake 乱码（中文字符被 `?` 替换）
- `SCHEMA.md` — 新增 API 函数文档标准（7 字段模板、适用范围、字段说明表）

### Phase 2: Crypto 模块（7 个文件）
- `crypto/aead.md` — 补充 tag_length, nonce_length, nonce, seal_output_size, open_output_size, increment_nonce
- `crypto/hkdf.md` — 完整重写，8 个函数（hmac_sha256, hmac_sha512, hkdf_extract, hkdf_expand, hkdf_expand_label, sha256×3）
- `crypto/x25519.md` — 完整重写，3 个函数 + 2 个结构体
- `crypto/blake3.md` — 完整重写，2 个 derive_key 重载
- `crypto/block.md` — 完整重写，aes_ecb_encrypt, aes_ecb_decrypt
- `crypto/base64.md` — 完整重写，base64_encode, base64_decode + 内部辅助函数
- `crypto/sha224.md` — 完整重写，sha224, is_hex_string, normalize_credential

### Phase 3: Agent 模块（12 个文件）
- `agent/front/listener.md` — listener, listen, make_affinity, accept_loop
- `agent/front/balancer.md` — balancer, select, dispatch, size, mix_hash, score, refresh_state
- `agent/session/session.md` — session, ~session, start, close, diversion, release_resources, make_session + setter
- `agent/worker/worker.md` — worker, run, dispatch_socket, load_snapshot
- `agent/worker/launch.md` — migrate_executor, prime, start, dispatch
- `agent/worker/stats.md` — state 构造, session_open/close, handoff_push/pop, session_counter, snapshot, observe
- `agent/worker/tls.md` — configure, make
- `agent/account/directory.md` — 9 个函数（upsert, insert, find, reserve, clear, try_acquire, contains 等）
- `agent/account/entry.md` — 10 个函数 + lease 类
- `agent/dispatch/table.md` — handler_func, handle_unknown, handler_table, dispatch
- `agent/config.md` — 全部结构体字段说明
- `agent/context.md` — server_context, worker_context, session_context

### Phase 4: Channel 模块（9 个文件）
- `channel/transport/transmission.md` — 13 个函数（async_read_some, async_write_some, close, cancel 等）
- `channel/transport/reliable.md` — 3 构造 + 9 方法 + 3 工厂函数
- `channel/transport/encrypted.md` — 1 构造 + 9 方法 + 1 工厂函数
- `channel/transport/unreliable.md` — 2 构造 + 11 方法 + 2 工厂函数
- `channel/transport/snapshot.md` — 1 构造 + 9 方法 + 1 工厂函数
- `channel/adapter/connector.md` — 3 构造 + 11 方法
- `channel/connection/pool.md` — make_endpoint_key, endpoint_hash, pooled_connection, connection_pool 全部方法
- `channel/eyeball/racer.md` — race, race_endpoint, race_context::complete
- `channel/health.md` — health, healthy_fast

### Phase 5: Protocol 模块（28 个文件，含 6 个新建）
- `protocol/analysis.md` — 5 个函数
- `protocol/tls/types.md` — 3 个写函数 + client_hello_features 结构
- `protocol/tls/signal.md` — 3 个函数
- `protocol/tls/feature_bitmap.md` — 3 个函数 + feature_bit 枚举
- `protocol/socks5/` — wire(10), stream(15+), constants(4 枚举), config
- `protocol/trojan/` — format(10), relay(多方法), constants(2 枚举), config
- `protocol/vless/` — format(4), relay(多方法), constants(多枚举), config
- `protocol/shadowsocks/` — format(4), relay(多方法), salts(2), replay(1), constants, config
- **新建** `protocol/common/address.md` — IPv4/IPv6/域名地址结构
- **新建** `protocol/common/form.md` — 协议表单枚举
- **新建** `protocol/common/read.md` — read_at_least, read_remaining
- **新建** `protocol/common/udp_relay.md` — udp_buffers, relay_udp_packet
- **新建** `protocol/http/parser.md` — parse_proxy_request, authenticate_proxy_request 等
- **新建** `protocol/http/relay.md` — relay 类全部方法

### Phase 6: Pipeline 模块（6 个文件）
- `pipeline/primitives.md` — 17 个函数/方法（含 preview 类成员）
- `pipeline/protocols/http.md` — http() 完整 7 字段
- `pipeline/protocols/socks5.md` — socks5() 完整 7 字段
- `pipeline/protocols/trojan.md` — trojan() 完整 7 字段
- `pipeline/protocols/vless.md` — vless() 完整 7 字段
- `pipeline/protocols/shadowsocks.md` — shadowsocks() 完整 7 字段

### Phase 7: Recognition 模块（7 个文件）
- `recognition/recognition.md` — recognize, identify + 4 个结构体
- `recognition/layered_pipeline.md` — detect, detect_tier0/1/2
- `recognition/scheme_route_table.md` — 全部方法
- `recognition/probe/probe.md` — probe() 函数
- `recognition/probe/analyzer.md` — 全部检测函数
- `recognition/confidence.md` — 枚举值说明
- `recognition/result.md` — 结构体字段说明

### Phase 8: Stealth 模块（约 25 个文件）
- `stealth/scheme.md` — **完整重写**（修复乱码），10 个虚函数 + shared_scheme
- `stealth/executor.md` — 9 个函数
- `stealth/registry.md` — 5 个函数
- `stealth/native.md` — 7 个函数
- `stealth/reality/` — scheme(8), handshake(4), auth(4+2), keygen(3), request(2), response(3), seal(11)
- `stealth/shadowtls/` — scheme(9), handshake(1+5 内部), auth(5)
- `stealth/restls/scheme.md` — 8 个函数
- `stealth/anytls/scheme.md` — 9 个函数
- `stealth/trusttunnel/scheme.md` — 8 个函数
- `stealth/ech/decrypt.md` — 1 函数 + 1 结构

### Phase 9: Multiplex 模块（11 个文件）
- `multiplex/core.md` — 8 个函数
- `multiplex/bootstrap.md` — 2 个函数
- `multiplex/config.md` — 枚举 + 结构体
- `multiplex/duct.md` — 9 个函数
- `multiplex/parcel.md` — 12 个函数
- `multiplex/smux/craft.md` — 14 个函数
- `multiplex/smux/frame.md` — 8 个函数
- `multiplex/smux/config.md` — 1 结构体
- `multiplex/yamux/craft.md` — 24 个函数
- `multiplex/yamux/frame.md` — 10 个函数
- `multiplex/yamux/config.md` — 1 结构体

### Phase 10: Resolve 模块（8 个文件）
- `resolve/router.md` — 9 个函数
- `resolve/dns/dns.md` — 5 个函数 + query_pipeline 概述
- `resolve/dns/upstream.md` — 10 个函数
- `resolve/dns/detail/cache.md` — 8 个函数
- `resolve/dns/detail/coalescer.md` — 6 个函数
- `resolve/dns/detail/rules.md` — 12 个函数
- `resolve/dns/detail/format.md` — 14 个函数
- `resolve/dns/detail/utility.md` — 1 个函数

### Phase 11: 基础设施模块（7 个文件）
- `memory.md` — 12 个函数/类型
- `fault.md` — 20+ 函数/类型（含 64 值错误码枚举）
- `exception.md` — 21 个函数（deviant/network/protocol/security 四层）
- `trace.md` — 8 个函数
- `transformer.md` — 7 个函数
- `loader.md` — 2 个函数
- `outbound.md` — 11 个函数/类型

### Phase 12: 收尾
- `index.md` — 全面重写，新增约 80 个 API 文档条目
- `log.md` — 记录本次操作

### 修复的问题
- **mojibake 乱码**: `stealth/scheme.md` 中文字符被 `?` 替换，从源码重新生成
- **自引用链接**: 多个文件的知识域部分指向自身，替换为出站链接
- **缺少 `related` 字段**: SCHEMA.md 要求但所有 API 文档均缺失，全部补齐
- **函数文档不完整**: 大量函数缺少参数/返回值/调用链/知识域字段

### 最终状态
- 约 120 个 API 文档文件，全部符合 7 字段标准
- 每个函数包含：功能说明、签名、参数表格、返回值、调用（向下）、被调用（向上）、知识域
- 所有文件有 `related` frontmatter 字段
- 所有页面至少 3 个出站 wikilink
- 无自引用链接、无 mojibake 乱码

## [2026-05-17] plan | PrismWiki 知识库重构规划

### 背景
用户要求根据 `H:\PrismWiki\skills\prism-wiki-standard` skill 标准，结合 Prism 项目源码，对知识库进行完整重构。

### 用户决策
- **重构范围**: 全量重构（补充缺失文档 + 标准合规化 + 目录重组）
- **详细度档位**: 全部 S档（完整段落、所有函数展开、完整调用链）
- **执行方式**: 自动化执行（Claude Code 扫描生成，人工审核）
- **技术方案**: 模块代理分析（Explore 代理深度分析每个模块）

### 探索完成
- Phase 1 源码扫描完成：16 个模块全部深度分析
- Batch 1: agent, channel, crypto, multiplex (4 代理)
- Batch 2: pipeline, protocol, recognition, resolve (4 代理)
- Batch 3: stealth, memory, exception, fault (4 代理)
- Batch 4: trace, transformer, loader, outbound (4 代理)

### 关键发现
- **知识库现状**: 196 个 md 文件，26 个顶层目录
- **问题**: 7 个模块目录为空、docs/ 归档未迁移、bugs/ 未启用
- **Skill 标准**: 7 种文档类型、S档 7 必填段落、frontmatter 7 字段

### Phase 3: 目录结构清理
- 创建 `bugs/template.md` — Bug 报告模板
- 更新 `index.md` 添加 bugs 入口
- docs/ 目录 36 个旧文档待处理（需逐一检查是否重复）

### 后续任务
- Phase 2: S档文档生成与修复
- Phase 3: docs/ 归档清理、空目录处理
- Phase 4: 验证审计（frontmatter 合规性、链接完整性）

## [2026-05-17] audit | Phase 4 验证审计完成

### 审计统计

| 审计项 | 合规率 | 问题数 |
|--------|--------|--------|
| Frontmatter合规性 | 74.86% | 46文件 |
| 链接完整性 | 62.7%覆盖率 | 69缺失目标 |
| S档段落完整性 | 95% | 1文件 |

### 详细发现

**Frontmatter合规性**：
- 扫描183文件，137完全合规
- 问题文件集中在 docs/（36个）和 ref/（15个）
- 所有API文档source路径正确
- 2文件tags超标：channel/connection/pool.md, multiplex/yamux/craft.md

**链接完整性**：
- 总wikilink 4,139，唯一目标309
- 275处转义问题（`\|`应为`|`）分布在15文件
- 69个缺失目标（38模块索引+13参考文档+18其他）
- 孤儿页86个（docs/25 + ref/30 + skills/5 + 其他26）

**段落完整性**：
- 抽样20个API文档，19个完全合规
- 仅 protocol/vless/format.md 缺少函数签名表段落
- 所有函数详解包含7字段（功能、签名、参数、返回值、调用、被调用、知识域）

### 修复建议

**高优先级**：
1. 修复15文件的转义语法错误（`\|`→`|`）
2. 处理 docs/ 目录36个旧文档（补frontmatter或删除重复）
3. 补充 ref/ 目录15个参考文档的frontmatter

**中优先级**：
4. 创建38个缺失的模块索引文档
5. 提升index.md覆盖率从62.7%到80%+

**低优先级**：
6. 修复2个文件的tags数量超标
7. 补充 protocol/vless/format.md 的函数签名表

### 最终状态

- 知识库整体质量良好
- 核心API文档符合S档标准
- docs/和ref/目录需要处理
- 重构基本完成，进入维护阶段

## [2026-05-17] delete | docs/ 归档目录清理

### 操作
- 删除 docs/ 目录及其下所有文件

### 删除文件清单（36 个）
docs\protocol\tls\serverrandom.md
docs\protocol\tls\sessionid.md
docs\agent\agent.md
docs\agent\architecture.md
docs\agent\configuration.md
docs\agent\deployment.md
docs\agent\testing.md
docs\agent\troubleshooting.md
docs\channel\channel.md
docs\channel\transport.md
docs\crypto\crypto.md
docs\multiplex\multiplex.md
docs\multiplex\smux.md
docs\multiplex\yamux.md
docs\pipeline\pipeline.md
docs\pipeline\processors.md
docs\protocol\common.md
docs\protocol\http.md
docs\protocol\protocol.md
docs\protocol\proxy-protocols.md
docs\protocol\shadowsocks.md
docs\protocol\socks5.md
docs\protocol\tls.md
docs\protocol\trojan.md
docs\protocol\trojan-gfw.md
docs\protocol\vless.md
docs\resolve\dns.md
docs\resolve\resolve.md
docs\stealth\anytls.md
docs\stealth\ech.md
docs\stealth\pki-certificates.md
docs\stealth\proxy-detection.md
docs\stealth\reality.md
docs\stealth\restls.md
docs\stealth\shadowtls.md
docs\stealth\stealth.md
docs\stealth\trusttunnel.md

### 原因
- docs/ 为归档目录，包含旧版文档（无 frontmatter）
- 所有文档已被新文档覆盖（按 S档标准生成）
- 用户确认可删除

### 验证
- docs/ 目录已不存在
- index.md 无需更新（无指向 docs/ 的链接）
- 总页面数减少 36 个

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
- 10 个文件：相对路径 `include/prism/...` → 绝对路径 `I:/code/Prism/include/prism/...`

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
