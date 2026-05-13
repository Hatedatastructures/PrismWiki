---
title: 测试体系
tags: [testing, ctest, unit, e2e, regression]
source:
  - I:/code/Prism/tests/ (47 个测试文件)
  - I:/code/Prism/tests/CMakeLists.txt
---

# 测试体系

Prism 包含 47 个独立测试可执行文件，覆盖基础设施、协议、加密、多路复用、伪装、识别、管道、DNS、端到端、并发和回归等维度。

## 运行方式

```bash
# 运行全部测试
ctest --test-dir build_release --output-on-failure

# 运行单个测试
build_release/tests/Session.exe
build_release/tests/Socks5.exe
```

每个测试是独立可执行文件，链接 `prism_static_library`。

---

## 基础设施

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| Fault | `Fault.cpp` | 错误码枚举、故障传播 |
| Exception | `Exception.cpp` | 异常层次结构（deviant→network/protocol/security） |
| FaultHandling | `FaultHandling.cpp` | 错误处理策略（热路径错误码 vs 启动异常） |
| MemoryArena | `MemoryArena.cpp` | 帧竞技场分配/重置 |
| Trace | `Trace.cpp` | 日志系统初始化和输出 |
| Config | `Config.cpp` | 配置文件加载和反序列化 |
| Connection | `Connection.cpp` | 连接池 acquire/recycle/cleanup |
| Session | `Session.cpp` | 会话生命周期管理 |
| Transmission | `Transmission.cpp` | 数据传输层 |
| Balancer | `Balancer.cpp` | 负载均衡器 |
| AccountDirectory | `AccountDirectory.cpp` | 账户目录（密码→SHA224、UUID 注册） |

---

## 协议

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| Http | `Http.cpp` | HTTP 代理请求解析（GET/CONNECT/POST）、转发请求行构建 |
| HttpParser | `HttpParser.cpp` | HTTP 解析器边界情况 |
| Socks5 | `Socks5.cpp` | SOCKS5 握手、UDP 头编解码、认证 |
| Trojan | `Trojan.cpp` | Trojan 凭据解析、UDP 包构建/解析 |
| Vless | `Vless.cpp` | VLESS 请求解析（IPv4/域名）、UDP 包、响应构建 |
| Shadowsocks | `Shadowsocks.cpp` | SS2022 地址端口解析、PSK 解码 |
| ProtocolToString | `ProtocolToString.cpp` | 协议类型到字符串转换 |

---

## 加密

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| Aead | `Aead.cpp` | AEAD (AES-128/256-GCM) 加解密正确性 |
| Blake3 | `Blake3.cpp` | BLAKE3 密钥派生 |
| Hkdf | `Hkdf.cpp` | HKDF Extract/Expand/ExpandLabel |
| X25519 | `X25519.cpp` | X25519 密钥生成、派生、交换 |
| Block | `Block.cpp` | 分组加密操作 |
| Base64Encode | `Base64Encode.cpp` | Base64 编解码 |
| Crypto | `Crypto.cpp` | 综合加密功能 |

---

## 多路复用

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| Smux | `Smux.cpp` | smux 帧序列化/反序列化 |
| SmuxCraft | `SmuxCraft.cpp` | smux 帧手工构造（SYN/PSH/FIN/NOP） |
| Yamux | `Yamux.cpp` | yamux 帧解析（Data/WindowUpdate/Ping/GoAway） |
| YamuxCraft | `YamuxCraft.cpp` | yamux 帧构造 |
| MultiplexDuct | `MultiplexDuct.cpp` | 多路复用管道（TCP 流/UDP parcel） |

---

## 伪装

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| Reality | `Reality.cpp` | Reality TLS 伪装方案 |
| Shadowtls | `Shadowtls.cpp` | ShadowTLS v2/v3 伪装 |
| ShadowTlsE2E | `ShadowTlsE2E.cpp` | ShadowTLS 端到端集成 |
| Restls | `Restls.cpp` | Restls 伪装方案 |
| Executor | `Executor.cpp` | 伪装方案执行器 |

---

## 识别

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| Recognition | `Recognition.cpp` | 协议识别三阶段流水线（probe→identify→dispatch） |
| FeatureBitmap | `FeatureBitmap.cpp` | TLS ClientHello 特征位图 |
| ProtocolAnalysis | `ProtocolAnalysis.cpp` | 协议特征分析 |
| SchemeRouteTable | `SchemeRouteTable.cpp` | 伪装方案路由表 |

---

## 管道

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| PipelinePrimitives | `PipelinePrimitives.cpp` | 隧道原语（双向透明转发） |

---

## DNS

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| DnsCache | `DnsCache.cpp` | DNS 缓存（LRU、TTL、过期淘汰） |
| DnsPacket | `DnsPacket.cpp` | DNS 报文构建/解析、IP 提取、TTL 计算 |
| DnsRules | `DnsRules.cpp` | DNS 规则引擎（地址映射、CNAME 重定向、否定应答） |

---

## 端到端

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| E2E | `E2E.cpp` | 完整代理链路端到端测试 |
| ShadowTlsE2E | `ShadowTlsE2E.cpp` | ShadowTLS 完整链路端到端测试 |

---

## 并发

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| ConcurrencyServer | `concurrency/server.cpp` | 并发服务端测试 |
| ConcurrencyClient | `concurrency/client.cpp` | 并发客户端测试 |

---

## 回归

| 测试 | 文件 | 测试内容 |
|------|------|----------|
| Regression | `Regression.cpp` | 历史 Bug 回归验证 |

---

## 相关链接

- [[Configuration]] — 配置参考
- [[Benchmark]] — 基准测试
- [[dev/stress]] — 压力测试
