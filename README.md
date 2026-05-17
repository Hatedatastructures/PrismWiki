# PrismWiki

Prism 高性能协程代理引擎的项目知识库 — 四层分离架构。

## 这是什么

PrismWiki 是 Prism 代理引擎的完整技术文档，采用四层分离架构，知识不重复原则。所有文档基于 Prism C++ 源码分析撰写。

**当前状态：** 389 个文档页面，四层架构（core/dev/docs/ref），覆盖 20+ 模块、25 种代理协议、6 种 TLS 伪装方案。

## 目录结构

```
PrismWiki/
├── core/                       # 模块实现层（详细）
│   ├── architecture.md         # 六阶段流水线架构
│   ├── startup.md              # 启动流程详解
│   ├── flow.md                 # 协议处理流程
│   ├── infrastructure.md       # 基础设施总览
│   ├── agent/                  # 前端监听、会话管理、负载均衡
│   ├── channel/                # 连接池、传输层、Happy Eyeballs
│   ├── pipeline/               # 协议处理器管道
│   ├── protocol/               # 协议编解码（SOCKS5/HTTP/Trojan/VLESS/Shadowsocks）
│   ├── stealth/                # TLS 伪装（Reality/ShadowTLS/Restls/AnyTLS/TrustTunnel/ECH）
│   ├── recognition/            # 协议智能识别
│   ├── resolve/                # DNS 解析、路由
│   ├── multiplex/              # 多路复用（Smux/Yamux）
│   ├── crypto/                 # 加密模块（AEAD/HKDF/X25519/Blake3）
│   ├── outbound/               # 出站代理
│   ├── memory/                 # PMR 内存池
│   ├── fault/                  # 错误码
│   ├── exception/              # 异常体系
│   ├── trace/                  # 日志系统
│   ├── transformer/            # JSON 序列化
│   └── loader/                 # 配置加载
│
├── dev/                        # 开发排障层（详细）
│   ├── coding/                 # 编码规范（命名/协程/PMR/注释/生命周期/错误处理）
│   ├── testing/                # 测试体系（框架/命令/基准/压力）
│   ├── debugging/              # 排障方法（连接/协议/内存/性能/TLS）
│   ├── building/               # 构建（CMake/依赖/命令/选项）
│   ├── performance/            # 性能优化
│   ├── extending/              # 扩展开发
│   ├── bugs/                   # Bug 记录
│   └ roadmap.md                # 开发路线图
│
├── docs/                       # 用户指南层（简单）
│   ├── getting-started.md      # 快速开始
│   ├── deployment.md           # 部署指南
│   ├── configuration.md        # 配置说明
│   ├── client-setup.md         # 客户端配置
│   ├── troubleshooting.md      # 故障排查
│   ├── faq.md                  # 常见问题
│   ├── upgrade.md              # 升级指南
│   ├── security.md             # 安全注意事项
│   └ performance-tips.md       # 性能建议
│
├── ref/                        # 参考知识层（中等）
│   ├── mihomo/                 # Mihomo 完整参考
│   │   ├── protocols/          # 25 种代理协议
│   │   ├── transport/          # 传输层插件
│   │   ├── mux/                # 多路复用
│   │   ├── proxy-groups/       # 代理组
│   │   ├── rules/              # 规则系统
│   │   ├── dns/                # DNS 配置
│   │   ├── tun/                # TUN 模式
│   │   ├── config/             # 配置模板（YAML）
│   │   ├── compatibility/      # Prism 兼容性
│   │   └ listeners/            # 入站监听
│   │   └ provider/             # Provider
│   │   └ sniffing/             # 协议嗅探
│   │   ├── ntp/                # NTP 同步
│   │   └ experimental/         # 实验性功能
│   │   └ script/               # 脚本规则
│   │   └ implementation/       # 实现参考
│   ├── crypto/                 # 加密知识（AEAD/HKDF/密钥交换）
│   ├── protocol/               # 协议知识（TLS/SOCKS5/HTTP/QUIC）
│   ├── network/                # 网络知识（Happy Eyeballs/DNS/GFW）
│   ├── programming/            # 编程知识（Boost.Asio/C++23 协程/PMR）
│   └ glossary.md               # 术语表
│
├── skills/                     # Claude Code Skills
│   └ prism-wiki-sync/          # 知识库同步检查
│
├── index.md                    # 总索引（Obsidian 入口）
├── SCHEMA.md                   # 知识库规范
├── log.md                      # 变更日志
└── README.md                   # 本文件
```

## 四层架构

| 层级 | 目录 | 详细程度 | 职责 |
|------|------|----------|------|
| 模块实现层 | core/ | 详细 | 函数实现、调用链、状态变化 |
| 开发排障层 | dev/ | 详细 | 编码规范、排障方法、Bug记录 |
| 用户指南层 | docs/ | 简单 | 用户文档、部署、配置 |
| 参考知识层 | ref/ | 中等 | 协议规范、加密原理、mihomo资料 |

## 知识不重复原则

同一知识点只在一处详细写，其他用 wikilink 引用：

- 函数实现细节 → core/
- 函数用途概述 → docs/ 或 ref/
- 相关原理 → ref/

示例：
- `core/stealth/reality/handshake.md` 详细写 Reality 握手实现
- `ref/mihomo/protocols/reality.md` 写 Reality 配置，引用实现文档

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
grep -rn "Reality" H:/PrismWiki/core/

# 查看模块概述
cat H:/PrismWiki/core/stealth/overview.md

# 查找 mihomo 协议配置
cat H:/PrismWiki/ref/mihomo/protocols/vless.md
```

## 文档规范

详见 `SCHEMA.md`：

- 四层分离架构，知识不重复
- YAML frontmatter：title, created, updated, layer, tags
- core 层必须标注 source（源码路径）
- 内部链接使用 Obsidian wikilink：`[[core/module/path|显示名]]`
- 复杂函数逐行解释，标注调用链

## 许可证

与 Prism 项目保持一致。