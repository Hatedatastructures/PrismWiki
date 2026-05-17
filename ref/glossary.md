---
title: Glossary
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [glossary, reference]
---

# 术语表

> Prism 知识库术语定义与解释。
> 最后更新：2026-05-17

---

## A

### AEAD
Authenticated Encryption with Associated Data。认证加密算法，同时提供加密和认证。如 AES-GCM、ChaCha20-Poly1305。详见 [[ref/crypto/aead-basics|AEAD 原理]]。

### Agent
Prism 前端模块，负责监听、会话管理、负载均衡。详见 [[core/agent/overview|Agent 模块]]。

### AnyTLS
TLS 伪装方案，使用 padding 技术隐藏 TLS 流量特征。详见 [[ref/mihomo/protocols/anytls|AnyTLS]] 和 [[core/stealth/anytls|AnyTLS 实现]]。

### Asio
Boost.Asio，C++ 异步 I/O 库。Prism 网络层基础。详见 [[ref/programming/boost-asio|Boost.Asio 协程]]。

---

## B

### Blake3
高速哈希算法，用于 Shadowsocks 2022。详见 [[ref/crypto/hkdf-theory|HKDF 理论]] 和 [[core/crypto/blake3|Blake3 实现]]。

### Buffer Pool
缓冲池，管理可复用内存块。详见 [[core/channel/overview|Channel 模块]]。

---

## C

### Channel
Prism 连接管理模块，负责连接池、传输层、Happy Eyeballs。详见 [[core/channel/overview|Channel 模块]]。

### C++23 协程
C++20/23 引入的协程特性，支持暂停/恢复执行。详见 [[ref/programming/cpp23-coroutine|C++23 协程]]。

### Constexpr
编译期计算，C++ 特性。详见 [[ref/programming/constexpr|Constexpr 计算]]。

### Coroutine
协程，可暂停恢复的函数。详见 [[ref/programming/cpp23-coroutine|C++23 协程]] 和 [[dev/coding/coroutine|协程约定]]。

---

## D

### DNS
Domain Name System。域名解析系统。详见 [[ref/protocol/dns-resolution|DNS 解析原理]] 和 [[core/resolve/dns|DNS 模块]]。

### Dispatch
六阶段流水线的第 3 阶段，负责会话分发。详见 [[core/architecture|六阶段流水线架构]]。

---

## E

### ECH
Encrypted Client Hello。TLS 扩展，加密握手信息。详见 [[ref/mihomo/protocols/ech|ECH]] 和 [[core/stealth/ech|ECH 实现]]。

### Error Code
错误码，整数标识错误类型。详见 [[core/fault/overview|Fault 模块]] 和 [[ref/programming/error-handling|错误处理策略]]。

---

## F

### Fault
Prism 错误码模块。详见 [[core/fault/overview|Fault 模块]]。

### Front
Agent 子模块，负责监听和前端接收。详见 [[core/agent/front/listener|Listener]]。

---

## G

### GFW
Great Firewall。网络审查系统。详见 [[ref/network/gfw|GFW 原理]]。

### Goroutine
Go 语言轻量级线程。详见 [[ref/programming/go-concurrency|Go 并发模型]]。

---

## H

### Happy Eyeballs
双栈连接算法，优先 IPv6 但快速 fallback IPv4。详见 [[ref/network/happy-eyeballs|Happy Eyeballs]] 和 [[core/channel/eyeball/racer|Racer 实现]]。

### HKDF
HMAC-based Key Derivation Function。密钥派生函数。详见 [[ref/crypto/hkdf-theory|HKDF 理论]] 和 [[core/crypto/hkdf|HKDF 实现]]。

### Hysteria2
基于 QUIC 的代理协议。详见 [[ref/mihomo/protocols/hysteria2|Hysteria2]]。

---

## I

### Incoming
六阶段流水线的第 1 阶段，负责接收连接。详见 [[core/architecture|六阶段流水线架构]]。

---

## K

### Key Exchange
密钥交换，如 X25519。详见 [[ref/crypto/key-exchange|密钥交换原理]] 和 [[core/crypto/x25519|X25519 实现]]。

---

## L

### Listener
监听器，接收新连接。详见 [[core/agent/front/listener|Listener]]。

### Loader
Prism 配置加载模块。详见 [[core/loader/overview|Loader 模块]]。

---

## M

### Mihomo
Go 语言代理客户端，原名 Clash.Meta。详见 [[ref/mihomo/overview|Mihomo 总索引]]。

### Monotonic Buffer
单向增长缓冲，PMR 资源类型。详见 [[ref/programming/pmr-concepts|PMR 概念]]。

### Multiplex
多路复用，在单连接上承载多个流。详见 [[core/multiplex/overview|Multiplex 模块]]。

---

## N

### Nonce
Number used once。加密算法的一次性值。详见 [[ref/crypto/aead-basics|AEAD 原理]]。

---

## O

### Outbound
六阶段流水线的第 6 阶段，负责出站代理。详见 [[core/architecture|六阶段流水线架构]] 和 [[core/outbound/overview|Outbound 模块]]。

---

## P

### Pipeline
六阶段流水线的第 4-5 阶段，协议处理器管道。详见 [[core/pipeline/overview|Pipeline 模块]]。

### PMR
Polymorphic Memory Resources。多态内存资源。详见 [[ref/programming/pmr-concepts|PMR 概念]] 和 [[dev/coding/pmr|PMR 使用规范]]。

### Pool Resource
池化内存资源，PMR 类型。详见 [[ref/programming/pmr-concepts|PMR 概念]]。

### Protocol
协议模块，定义协议格式和常量。详见 [[core/protocol/overview|Protocol 模块]]。

---

## Q

### QUIC
Quick UDP Internet Connections。基于 UDP 的传输协议。详见 [[ref/protocol/quic-basics|QUIC 基础]]。

---

## R

### Recognition
六阶段流水线的第 2 阶段，协议识别。详见 [[core/recognition/overview|Recognition 模块]]。

### Reality
TLS 伪装方案，使用真实 TLS 服务器握手特征。详见 [[ref/mihomo/protocols/reality|Reality]] 和 [[core/stealth/reality/handshake|Reality 握手实现]]。

### Restls
TLS 伪装方案，模拟 TLS 响应时序。详见 [[ref/mihomo/transport/restls|Restls]] 和 [[core/stealth/restls|Restls 实现]]。

### Resolve
DNS 解析模块。详见 [[core/resolve/overview|Resolve 模块]]。

---

## S

### Session
会话，代表一个代理连接上下文。详见 [[core/session/session|Session]]。

### Shadowsocks
代理协议。2022 版本使用 Blake3。详见 [[ref/mihomo/protocols/shadowsocks|Shadowsocks]] 和 [[core/protocol/shadowsocks|Shadowsocks 实现]]。

### ShadowTLS
TLS 伪装方案，使用真实 TLS 握手包装代理协议。详见 [[ref/mihomo/transport/shadowtls|ShadowTLS]] 和 [[core/stealth/shadowtls|ShadowTLS 实现]]。

### Smux
简单多路复用协议。详见 [[ref/mihomo/mux/smux|Smux]] 和 [[core/multiplex/smux/craft|Smux 实现]]。

### SOCKS5
代理协议标准。详见 [[ref/mihomo/protocols/socks5|SOCKS5]] 和 [[core/protocol/socks5|SOCKS5 实现]]。

### Stealth
TLS 伪装模块。详见 [[core/stealth/overview|Stealth 模块]]。

### Strand
Asio 序列化执行器。详见 [[ref/programming/boost-asio|Boost.Asio 协程]]。

---

## T

### TCP
Transmission Control Protocol。传输控制协议。详见 [[ref/protocol/tcp-basics|TCP 基础]]。

### TLS
Transport Layer Security。传输层安全协议。详见 [[ref/protocol/tls-handshake|TLS 握手流程]]。

### Trace
Prism 日志模块。详见 [[core/trace/overview|Trace 模块]]。

### Trojan
代理协议，使用 TLS 承载。详见 [[ref/mihomo/protocols/trojan|Trojan]] 和 [[core/protocol/trojan|Trojan 实现]]。

### TrustTunnel
TLS 伪装方案，使用自定义 TLS 扩展。详见 [[ref/mihomo/protocols/trusttunnel|TrustTunnel]] 和 [[core/stealth/trusttunnel|TrustTunnel 实现]]。

### TUIC
基于 QUIC 的代理协议。详见 [[ref/mihomo/protocols/tuic|TUIC]]。

### TUN
虚拟网络接口模式。详见 [[ref/mihomo/tun/overview|TUN 模式]]。

---

## V

### VLESS
轻量代理协议。详见 [[ref/mihomo/protocols/vless|VLESS]] 和 [[core/protocol/vless|VLESS 实现]]。

### VMess
V2Ray 协议。详见 [[ref/mihomo/protocols/vmess|VMess]]。

---

## W

### WireGuard
VPN 协议。详见 [[ref/mihomo/protocols/wireguard|WireGuard]]。

---

## X

### X25519
Curve25519 密钥交换。详见 [[ref/crypto/key-exchange|密钥交换原理]] 和 [[core/crypto/x25519|X25519 实现]]。

---

## Y

### Yamux
多路复用协议。详见 [[ref/mihomo/mux/yamux|Yamux]] 和 [[core/multiplex/yamux/craft|Yamux 实现]]。

---

## 数字

### 2022-blake3
Shadowsocks 2022 方法名，使用 Blake3。详见 [[ref/mihomo/protocols/shadowsocks|Shadowsocks]]。

---

## 相关参考

- [[ref/crypto/overview|加密知识]]
- [[ref/protocol/overview|协议知识]]
- [[ref/network/overview|网络知识]]
- [[ref/programming/overview|编程知识]]
- [[ref/mihomo/overview|Mihomo 参考]]