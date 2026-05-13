# PrismWiki

Prism 高性能协程代理引擎的项目知识库。

## 这是什么

PrismWiki 是 Prism 代理引擎的完整技术文档，覆盖模块设计、协议实现、伪装方案、客户端对接、调试排障等所有方面。所有文档基于 Prism C++ 源码分析撰写，不是泛泛的协议介绍。

**当前状态：** 67 个文档页面，覆盖 16 个核心模块 + 5 个伪装方案 + 6 个代理协议 + 客户端对接 + 开发笔记。

## 目录结构

```
PrismWiki/
├── agent/                  # 前端监听、会话管理、负载均衡
│   ├── agent.md            # 模块概述 + 详细设计
│   ├── architecture.md     # 分层架构
│   ├── configuration.md    # 配置详解
│   ├── deployment.md       # 部署指南
│   ├── testing.md          # 测试体系
│   └── troubleshooting.md  # 故障排查
├── channel/                # 连接池、传输层抽象
│   ├── channel.md
│   └── transport.md        # 传输层接口分层
├── crypto/                 # AEAD、Blake3、X25519、HKDF
│   ├── crypto.md
│   └── aead.md             # AEAD 加密实现
├── memory/                 # PMR 内存池
│   ├── memory.md
│   └── pool.md             # 三级内存池架构
├── multiplex/              # 多路复用
│   ├── multiplex.md
│   ├── smux.md             # smux 协议规范
│   └── yamux.md            # yamux 协议规范
├── protocol/               # 协议编解码
│   ├── protocol.md         # 模块概述
│   ├── common.md           # 公共组件
│   ├── tls.md              # TLS 特征分析
│   ├── http.md             # HTTP 代理协议
│   ├── socks5.md           # SOCKS5 协议
│   ├── trojan.md           # Trojan 协议
│   ├── trojan-gfw.md       # Trojan-GFW 原始实现
│   ├── vless.md            # VLESS 协议
│   ├── shadowsocks.md      # Shadowsocks 2022
│   └── proxy-protocols.md  # 协议总览与选型
├── pipeline/               # 协议处理管道
│   ├── pipeline.md
│   └── processors.md       # 协议处理器详解
├── recognition/            # 协议智能识别
│   └── recognition.md
├── resolve/                # DNS 解析、路由
│   ├── resolve.md
│   └── dns.md              # DNS 解析器管道
├── stealth/                # TLS 伪装方案
│   ├── stealth.md          # 模块概述
│   ├── reality.md          # Reality
│   ├── shadowtls.md        # ShadowTLS
│   ├── restls.md           # Restls
│   ├── anytls.md           # AnyTLS
│   ├── trusttunnel.md      # TrustTunnel
│   ├── ech.md              # ECH
│   ├── native.md           # 原生 TLS fallback
│   ├── pki-certificates.md # PKI 证书体系
│   └── proxy-detection.md  # 代理检测技术
├── exception/              # 异常层次结构
├── fault/                  # 错误码枚举
├── trace/                  # spdlog 日志
├── transformer/            # glaze JSON 序列化
├── loader/                 # 配置加载
├── outbound/               # 出站代理接口
├── client/                 # mihomo 客户端对接
├── dev/                    # 开发笔记、配置参考、测试体系
├── performance/            # 基准测试报告
├── index.md                # 分类索引
├── SCHEMA.md               # 文档规范
└── log.md                  # 变更日志
```

## 怎么用

### 作为 Obsidian 知识库

1. 克隆本仓库
2. 用 Obsidian 打开目录
3. 通过 `index.md` 浏览，或用 Obsidian 图谱探索关联

### 作为 Prism 子模块

```bash
cd I:/code/Prism
git submodule add <wiki-repo-url> docs/wiki
```

### 命令行查询

```bash
# 搜索关键词
grep -rn "Happy Eyeballs" H:/wiki/

# 查看模块概述
cat H:/wiki/channel/channel.md

# 查找所有提到某个协议的文件
grep -rl "Reality" H:/wiki/
```

## 文档规范

- 每个模块一个文件夹，主文件名为 `模块名.md`
- YAML frontmatter：title, created, updated, type, tags, related
- 内部链接使用 Obsidian wikilink：`[[模块名]]`
- 概述和详细设计合并在一个文件中，不拆分
- 基于源码分析撰写，不是照抄官方文档

## 许可证

与 Prism 项目保持一致。
