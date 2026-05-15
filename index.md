# Prism 知识库

> Prism 高性能代理引擎的项目文档 — 覆盖模块设计、实现细节、调试排障、客户端对接、Bug 记录。
> 最后更新：2026-05-15

---

## 核心模块

- [[memory]] — PMR 内存池、三级池化、帧分配器
- [[fault]] — 错误码枚举、双轨错误处理
- [[exception]] — 异常层次结构
- [[trace]] — spdlog 异步日志
- [[transformer]] — glaze JSON 序列化
- [[loader]] — 配置加载、账户目录构建
- [[outbound]] — 出站代理接口、直连实现

## Agent 模块

- [[agent/config]] — Agent 运行时配置类型定义
- [[agent/context]] — Agent 运行时上下文类型定义
- [[agent/front/listener]] — 前端监听器
- [[agent/front/balancer]] — 负载均衡器
- [[agent/session/session]] — 会话生命周期管理
- [[agent/worker/worker]] — Worker 线程核心实现
- [[agent/worker/launch]] — 会话启动与连接分发
- [[agent/worker/stats]] — 负载统计
- [[agent/worker/tls]] — TLS 上下文初始化
- [[agent/account/directory]] — 账户目录
- [[agent/account/entry]] — 账户条目与租约
- [[agent/dispatch/table]] — 协议处理器分发表

## Channel 模块

- [[channel/transport/transmission]] — 传输层抽象接口
- [[channel/transport/reliable]] — TCP 可靠传输
- [[channel/transport/encrypted]] — TLS 加密传输
- [[channel/transport/unreliable]] — UDP 不可靠传输
- [[channel/transport/snapshot]] — 预读快照传输
- [[channel/adapter/connector]] — TLS 连接器
- [[channel/connection/pool]] — 连接池
- [[channel/eyeball/racer]] — Happy Eyeballs 并行连接
- [[channel/health]] — 连接健康检查

## Crypto 模块

- [[crypto/aead]] — AEAD 加密（AES-GCM / ChaCha20-Poly1305）
- [[crypto/hkdf]] — HKDF 密钥派生 + HMAC
- [[crypto/x25519]] — X25519 密钥交换
- [[crypto/blake3]] — Blake3 哈希
- [[crypto/block]] — AES-ECB 块加密
- [[crypto/base64]] — Base64 编解码
- [[crypto/sha224]] — SHA-224 哈希

## Pipeline 模块

- [[pipeline/primitives]] — 协议管道原语操作
- [[pipeline/protocols/http]] — HTTP 协议处理器
- [[pipeline/protocols/socks5]] — SOCKS5 协议处理器
- [[pipeline/protocols/trojan]] — Trojan 协议处理器
- [[pipeline/protocols/vless]] — VLESS 协议处理器
- [[pipeline/protocols/shadowsocks]] — Shadowsocks 协议处理器

## Protocol 模块

- [[protocol/analysis]] — 协议分析与检测
- [[protocol/tls/types]] — TLS 类型定义与常量
- [[protocol/tls/signal]] — TLS 信号检测
- [[protocol/tls/feature_bitmap]] — TLS 特征位图

### 通用组件

- [[protocol/common/address]] — 地址解析（IPv4/IPv6/域名）
- [[protocol/common/form]] — 协议表单枚举
- [[protocol/common/read]] — 协议读取工具
- [[protocol/common/udp_relay]] — UDP 中继公共组件

### HTTP

- [[protocol/http/parser]] — HTTP 代理请求解析
- [[protocol/http/relay]] — HTTP 中继

### SOCKS5

- [[protocol/socks5/wire]] — SOCKS5 线格式编解码
- [[protocol/socks5/stream]] — SOCKS5 流处理
- [[protocol/socks5/constants]] — SOCKS5 常量与枚举
- [[protocol/socks5/config]] — SOCKS5 配置

### Trojan

- [[protocol/trojan/format]] — Trojan 帧格式
- [[protocol/trojan/relay]] — Trojan 中继
- [[protocol/trojan/constants]] — Trojan 常量与枚举
- [[protocol/trojan/config]] — Trojan 配置

### VLESS

- [[protocol/vless/format]] — VLESS 帧格式
- [[protocol/vless/relay]] — VLESS 中继
- [[protocol/vless/constants]] — VLESS 常量与枚举
- [[protocol/vless/config]] — VLESS 配置

### Shadowsocks

- [[protocol/shadowsocks/format]] — SS2022 帧格式
- [[protocol/shadowsocks/relay]] — SS2022 中继
- [[protocol/shadowsocks/salts]] — SS2022 盐值管理
- [[protocol/shadowsocks/replay]] — SS2022 重放检测
- [[protocol/shadowsocks/constants]] — SS2022 常量
- [[protocol/shadowsocks/config]] — SS2022 配置

## Recognition 模块

- [[recognition/recognition]] — 协议识别入口
- [[recognition/layered_pipeline]] — 分层检测管道
- [[recognition/scheme_route_table]] — 方案路由表
- [[recognition/confidence]] — 置信度枚举
- [[recognition/result]] — 识别结果结构
- [[recognition/probe/probe]] — 协议探测
- [[recognition/probe/analyzer]] — 特征分析器

## Resolve 模块

- [[resolve/router]] — DNS 路由器
- [[resolve/dns/dns]] — DNS 解析入口
- [[resolve/dns/upstream]] — 上游 DNS 查询
- [[resolve/dns/config]] — DNS 配置
- [[resolve/dns/detail/cache]] — DNS 缓存
- [[resolve/dns/detail/coalescer]] — 请求合并
- [[resolve/dns/detail/rules]] — 域名规则匹配
- [[resolve/dns/detail/format]] — DNS 报文格式
- [[resolve/dns/detail/utility]] — DNS 工具函数

## Multiplex 模块

- [[multiplex/core]] — 多路复用核心
- [[multiplex/bootstrap]] — 协商启动
- [[multiplex/duct]] — TCP 流管道
- [[multiplex/parcel]] — UDP 数据报管道
- [[multiplex/config]] — 多路复用配置
- [[multiplex/smux/craft]] — smux 帧编解码
- [[multiplex/smux/frame]] — smux 帧格式
- [[multiplex/smux/config]] — smux 配置
- [[multiplex/yamux/craft]] — yamux 帧编解码
- [[multiplex/yamux/frame]] — yamux 帧格式
- [[multiplex/yamux/config]] — yamux 配置

## Stealth 模块

- [[stealth/scheme]] — 伪装方案基类（分层检测架构）
- [[stealth/executor]] — 方案执行器
- [[stealth/registry]] — 方案注册表
- [[stealth/native]] — 原生 TLS（无伪装 fallback）

### Reality

- [[stealth/reality/scheme]] — Reality 方案
- [[stealth/reality/handshake]] — Reality 握手
- [[stealth/reality/auth]] — Reality 认证
- [[stealth/reality/keygen]] — Reality 密钥派生
- [[stealth/reality/request]] — Reality 请求解析
- [[stealth/reality/response]] — Reality 响应生成
- [[stealth/reality/seal]] — Reality 加密封装
- [[stealth/reality/config]] — Reality 配置
- [[stealth/reality/constants]] — Reality 常量

### ShadowTLS

- [[stealth/shadowtls/scheme]] — ShadowTLS 方案
- [[stealth/shadowtls/handshake]] — ShadowTLS 握手
- [[stealth/shadowtls/auth]] — ShadowTLS HMAC 认证
- [[stealth/shadowtls/config]] — ShadowTLS 配置
- [[stealth/shadowtls/constants]] — ShadowTLS 常量

### 其他伪装方案

- [[stealth/restls/scheme]] — Restls 方案
- [[stealth/restls/config]] — Restls 配置
- [[stealth/anytls/scheme]] — AnyTLS 方案
- [[stealth/anytls/config]] — AnyTLS 配置
- [[stealth/trusttunnel/scheme]] — TrustTunnel 方案
- [[stealth/trusttunnel/config]] — TrustTunnel 配置
- [[stealth/ech/decrypt]] — ECH 解密
- [[stealth/ech/config]] — ECH 配置

## 客户端对接

- [[client/mihomo-meta]] — mihomo 客户端概述
- [[client/mihomo-clash-config]] — mihomo 配置详解
- [[client/mihomo-proxy-groups]] — 代理组配置
- [[client/mihomo-rules]] — 规则系统
- [[client/mihomo-dns]] — DNS 处理机制
- [[client/tun]] — TUN 模式

## 开发笔记

- [[dev/modules]] — Prism 全局模块结构
- [[dev/cpp23-coroutines]] — C++23 协程
- [[dev/pmr-memory-pool]] — PMR 内存池
- [[dev/tcp]] — TCP 协议基础
- [[dev/tls]] — TLS 协议基础
- [[dev/udp]] — UDP 协议基础
- [[dev/gfw]] — GFW 封锁原理
- [[dev/configuration]] — 完整配置文件参考
- [[dev/testing]] — 测试体系
- [[dev/stress]] — 压力测试

## 性能报告

- [[performance/report]] — 基准测试报告
- [[performance/benchmark]] — 基准测试详解（12 个 bench）

## bugs/ — Bug 记录

> 暂无记录。
