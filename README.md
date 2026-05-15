# PrismWiki

Prism 高性能协程代理引擎的项目知识库。

## 这是什么

PrismWiki 是 Prism 代理引擎的完整技术文档，覆盖模块设计、协议实现、伪装方案、客户端对接、调试排障等所有方面。所有文档基于 Prism C++ 源码分析撰写，不是泛泛的协议介绍。

**当前状态：** 196 个文档页面，覆盖 16 个核心模块 + 7 种伪装方案 + 6 个代理协议 + 客户端对接 + 开发笔记 + 39 篇技术参考。

## 目录结构

```
PrismWiki/
├── agent/                      # 前端监听、会话管理、负载均衡
│   ├── config.md               # 运行时配置
│   ├── context.md              # 运行时上下文
│   ├── front/                  # 监听器 + 负载均衡器
│   ├── session/                # 会话生命周期
│   ├── worker/                 # Worker 线程、启动、统计、TLS
│   ├── account/                # 账户目录与条目
│   └── dispatch/               # 协议处理器分发表
├── channel/                    # 连接池、传输层抽象
│   ├── transport/              # 传输层接口（可靠/加密/不可靠/快照）
│   ├── adapter/                # TLS 连接器
│   ├── connection/             # 连接池
│   ├── eyeball/                # Happy Eyeballs 并行连接
│   └── health.md               # 连接健康检查
├── crypto/                     # AEAD、Blake3、X25519、HKDF、Base64
├── memory/                     # PMR 内存池
├── multiplex/                  # 多路复用核心 + smux/yamux 实现
├── pipeline/                   # 协议处理管道 + 各协议处理器
├── protocol/                   # 协议编解码
│   ├── common/                 # 地址解析、表单、读取工具
│   ├── tls/                    # TLS 类型、信号、特征位图
│   ├── http/                   # HTTP 代理
│   ├── socks5/                 # SOCKS5 协议
│   ├── trojan/                 # Trojan 协议
│   ├── vless/                  # VLESS 协议
│   ├── shadowsocks/            # Shadowsocks 2022
│   └── analysis.md             # 协议分析与检测
├── recognition/                # 协议智能识别（分层管道、置信度、探测）
├── resolve/                    # DNS 路由 + DNS 解析器（缓存/合并/规则）
├── stealth/                    # TLS 伪装方案
│   ├── reality/                # Reality（握手/认证/密钥/封装）
│   ├── shadowtls/              # ShadowTLS
│   ├── restls/                 # Restls
│   ├── anytls/                 # AnyTLS
│   ├── trusttunnel/            # TrustTunnel
│   ├── ech/                    # ECH 解密
│   ├── native.md               # 原生 TLS fallback
│   ├── scheme.md               # 伪装方案基类
│   ├── executor.md             # 方案执行器
│   └── registry.md             # 方案注册表
├── exception.md                # 异常层次结构
├── fault.md                    # 错误码枚举
├── trace.md                    # spdlog 日志
├── transformer.md              # glaze JSON 序列化
├── loader.md                   # 配置加载
├── outbound.md                 # 出站代理接口
├── client/                     # mihomo 客户端对接
├── dev/                        # 开发笔记（协程、TCP/UDP/TLS、配置、测试）
├── ref/                        # 技术参考文档
│   ├── anti-censorship/        # DPI、中间人劫持、探测、指纹、流量分析
│   ├── crypto/                 # AES-GCM、ChaCha20、ECDHE、HKDF 等
│   ├── memory/                 # Arena、PMR、零拷贝
│   ├── network/                # 连接池、Happy Eyeballs、NAT、TCP、UDP
│   ├── programming/            # Boost.Asio、C++23 协程、constexpr
│   └── protocol/               # DNS over X、HTTP CONNECT、TLS 1.3、SOCKS5 RFC
├── docs/                       # 旧版文档归档
├── index.md                    # 分类索引（Obsidian 入口）
├── SCHEMA.md                   # 文档规范
└── log.md                      # 变更日志
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
grep -rn "Happy Eyeballs" H:/PrismWiki/

# 查看模块概述
cat H:/PrismWiki/channel/health.md

# 查找所有提到某个协议的文件
grep -rl "Reality" H:/PrismWiki/
```

## 文档规范

- 每个模块一个文件夹，按职责拆分为多个细粒度文件
- YAML frontmatter：title, created, updated, type, tags, related
- 内部链接使用 Obsidian wikilink：`[[模块名]]`
- 基于源码分析撰写，不是照抄官方文档
- `ref/` 目录存放底层技术参考，与 Prism 实现解耦

## 许可证

与 Prism 项目保持一致。
