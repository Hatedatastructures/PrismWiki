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
- gfw.md: [[TLS-指纹]] → [[tls-fingerprint]], [[Traffic-Analysis]] → [[流量分析与对抗]]
- traffic-analysis.md: [[TLS-指纹]] → [[tls-fingerprint]] (2处)
- tls-camouflage-comparison.md: [[Traffic-Analysis]] → [[流量分析与对抗]]
- mihomo-meta.md: [[TUN]] → [[TUN-模式]]

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
- prism-architecture: 添加 [[TCP]], [[TLS]], [[mihomo-Meta]]
- prism-configuration: 添加 [[Mihomo-Clash-Config]], [[mihomo-Meta]]
- prism-deployment: 添加 [[mihomo-Meta]], [[GFW]]
- prism-modules: 添加 [[mihomo-Meta]], [[Proxy-Protocols]]
- prism-testing: 添加 [[TCP]], [[Proxy-Protocols]]
- prism-troubleshooting: 添加 [[mihomo-Meta]], [[GFW]], [[DNS]]
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
