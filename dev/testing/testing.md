---
layer: dev
title: 测试体系完整指南
source:
  - tests/ (47 个测试文件)
  - tests/CMakeLists.txt
  - tests/common/TestRunner.hpp
module: testing
type: reference
tags: [testing, ctest, unit, e2e, regression, integration, protocol, crypto]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing/stress|stress]]"
  - "[[core/overview|overview]]"
  - "[[dev/building/configuration|configuration]]"
  - "[[core/architecture|architecture]]"
layer: dev
---

# 测试体系完整指南

测试是软件质量的基石。Prism 采用独立可执行文件测试架构，包含 47 个测试覆盖基础设施、协议、加密、多路复用、伪装、识别、管道、DNS、端到端、并发和回归等维度。

## 概述

### 测试架构设计

Prism 的测试体系遵循以下设计原则：

```
+--------------------------------------------------+
|                    测试入口                       |
|              ctest --test-dir build              |
+--------------------------------------------------+
                       |
      +----------------+----------------+
      |                |                |
+----------+    +------------+    +------------+
| 单元测试 |    | 集成测试   |    | E2E测试   |
+----------+    +------------+    +------------+
      |                |                |
      +----+----+      +----+----+      +----+
           |                |                |
    +------+------+  +------+------+  +------+------+
    |基础设施测试|  |协议集成测试|  |并发测试    |
    +------+------+  +------+------+  +------+------+
           |                |                |
    +------+------+  +------+------+        |
    |加密模块测试|  |伪装模块测试|        |
    +------+------+  +------+------+        |
           |                |                |
    +------+------+  +------+------+        |
    |多路复用测试|  |识别模块测试|        |
    +------+------+  +------+------+        |
           |                                     |
    +------+------+                        +----+
    |DNS模块测试 |                        |回归|
    +------+------+                        +----+
```

每个测试是独立可执行文件的优势：

| 优势 | 说明 |
|------|------|
| 隔离性 | 测试失败不影响其他测试 |
| 并行性 | 多个测试可同时运行 |
| 调试性 | 单独运行便于问题定位 |
| 可定制 | 每个测试可独立配置 |

### 测试分类总览

Prism 的 47 个测试按模块分类：

| 分类 | 测试数 | 覆盖范围 |
|------|--------|----------|
| 基础设施 | 12 | Fault, Exception, Memory, Trace, Config, Connection, Session, Transmission, Balancer, AccountDirectory |
| 协议 | 7 | Http, HttpParser, Socks5, Trojan, Vless, Shadowsocks, ProtocolToString |
| 加密 | 8 | Aead, Blake3, Hkdf, X25519, Block, Base64Encode, Crypto |
| 多路复用 | 5 | Smux, SmuxCraft, Yamux, YamuxCraft, MultiplexDuct |
| 伪装 | 5 | Reality, Shadowtls, ShadowTlsE2E, Restls, Executor |
| 识别 | 4 | Recognition, FeatureBitmap, ProtocolAnalysis, SchemeRouteTable |
| 管道 | 1 | PipelinePrimitives |
| DNS | 3 | DnsCache, DnsPacket, DnsRules |
| 端到端 | 2 | E2E, ShadowTlsE2E |
| 并发 | 2 | ConcurrencyServer, ConcurrencyClient |
| 回归 | 1 | Regression |

### TestRunner 测试框架

所有测试共用 `tests/common/TestRunner.hpp` 中的统一框架：

```cpp
/**
 * @brief 测试运行器基类
 * @details 提供统一的测试结果记录和汇总输出
 */
class TestRunner {
public:
    /**
     * @brief 构造测试运行器
     * @param tag 测试模块标签，用于区分输出
     */
    explicit TestRunner(std::string_view tag);
    
    /**
     * @brief 检查条件并记录结果
     * @param condition 待检查的条件
     * @param message 结果描述信息
     */
    void Check(bool condition, std::string_view message);
    
    /**
     * @brief 手动记录成功
     * @param message 成功描述
     */
    void LogPass(std::string_view message);
    
    /**
     * @brief 手动记录失败
     * @param message 失败描述
     */
    void LogFail(std::string_view message);
    
    /**
     * @brief 输出结果汇总并返回退出码
     * @return 0 = 全部通过，1 = 存在失败
     */
    int Summary();
    
private:
    std::string tag_;
    int pass_count_ = 0;
    int fail_count_ = 0;
};
```

### 运行方式

```bash
# 运行全部测试（CTest）
ctest --test-dir build_release --output-on-failure

# 运行单个测试
build_release/tests/Session.exe
build_release/tests/Socks5.exe
build_release/tests/Crypto.exe

# 运行指定标签的测试
ctest --test-dir build_release -L protocol

# 并行运行测试（4 线程）
ctest --test-dir build_release -j 4
```

---

## 详解

### 基础设施测试

基础设施测试验证核心系统组件的正确性。

#### Fault — 错误码系统

**文件**: `tests/Fault.cpp`

测试内容：
- 错误码枚举完整性
- 错误码到字符串转换
- 错误传播机制

```cpp
// Fault 测试示例
void TestFaultCodes() {
    TestRunner runner("Fault");
    
    // 测试错误码枚举
    runner.Check(fault::code::success == 0, "success is 0");
    runner.Check(fault::code::network_error > 0, "network_error > 0");
    
    // 测试字符串转换
    auto str = fault::to_string(fault::code::timeout);
    runner.Check(str == "timeout", "to_string works");
    
    runner.Summary();
}
```

#### Exception — 异常层次

**文件**: `tests/Exception.cpp`

测试 Prism 的异常层次结构：

```
deviant (基类)
  ├── network    — 网络异常
  ├── protocol   — 协议异常
  └── security   — 安全异常
```

测试内容：
- 异常类型继承关系
- 异常消息格式
- 异常捕获和传播

#### MemoryArena — 帧竞技场

**文件**: `tests/MemoryArena.cpp`

测试内容：
- 帧内分配正确性
- reset 后内存释放
- 多次分配不重叠

```cpp
// MemoryArena 测试示例
void TestArenaAllocation() {
    TestRunner runner("MemoryArena");
    
    frame_arena arena;
    
    // 分配多个对象
    auto* p1 = arena.allocate(100);
    auto* p2 = arena.allocate(200);
    
    // 验证地址不重叠
    runner.Check(p1 != p2, "different addresses");
    runner.Check((char*)p2 >= (char*)p1 + 100, "no overlap");
    
    // 重置后可重新分配
    arena.reset();
    auto* p3 = arena.allocate(100);
    runner.Check(p3 == p1, "reuse after reset");
    
    runner.Summary();
}
```

#### Connection — 连接池

**文件**: `tests/Connection.cpp`

测试内容：
- acquire 获取连接
- recycle 彔回连接
- cleanup 清理过期连接
- 僵尸检测

#### Session — 会话管理

**文件**: `tests/Session.cpp`

测试内容：
- 会话创建和销毁
- 会话状态管理
- 协程生命周期绑定

#### Balancer — 负载均衡

**文件**: `tests/Balancer.cpp`

测试内容：
- worker 选择算法
- 亲和性哈希计算
- 负载分布均衡性

#### AccountDirectory — 账户目录

**文件**: `tests/AccountDirectory.cpp`

测试内容：
- 密码到 SHA224 转换
- UUID 注册和查询
- 用户认证流程

---

### 协议测试

协议测试验证各代理协议的解析和构建正确性。

#### Http — HTTP 代理

**文件**: `tests/Http.cpp`

测试内容：
- GET 请求解析
- CONNECT 请求解析
- POST 请求解析
- 请求行构建
- 响应解析

```cpp
// Http 测试示例
void TestHttpRequest() {
    TestRunner runner("Http");
    
    // 解析 GET 请求
    std::string request = "GET http://example.com/ HTTP/1.1\r\n"
                          "Host: example.com\r\n\r\n";
    
    auto parsed = http::parse_request(request);
    runner.Check(parsed.method == "GET", "method is GET");
    runner.Check(parsed.url == "http://example.com/", "url parsed");
    runner.Check(parsed.version == "HTTP/1.1", "version parsed");
    
    runner.Summary();
}
```

#### Socks5 — SOCKS5 代理

**文件**: `tests/Socks5.cpp`

测试内容：
- 版本协商
- 用户名/密码认证
- CONNECT 命令处理
- UDP ASSOCIATE 命令
- UDP 头编解码

```cpp
// Socks5 UDP 头测试
void TestUdpHeader() {
    TestRunner runner("Socks5");
    
    // 构建 UDP 头
    socks5::udp_header header;
    header.frag = 0;
    header.address = "192.168.1.1";
    header.port = 443;
    
    auto encoded = header.encode();
    
    // 解析验证
    auto decoded = socks5::parse_udp_header(encoded);
    runner.Check(decoded.frag == 0, "frag matches");
    runner.Check(decoded.address == "192.168.1.1", "address matches");
    runner.Check(decoded.port == 443, "port matches");
    
    runner.Summary();
}
```

#### Trojan — Trojan 协议

**文件**: `tests/Trojan.cpp`

测试内容：
- 凭据解析（SHA224 密码）
- TCP 请求构建
- UDP 包构建/解析

#### Vless — VLESS 协议

**文件**: `tests/Vless.cpp`

测试内容：
- UUID 解析
- IPv4 地址请求
- 域名地址请求
- UDP 包格式
- 响应构建

#### Shadowsocks — SS2022 协议

**文件**: `tests/Shadowsocks.cpp`

测试内容：
- PSK Base64 解码
- 16B/32B 密钥（AES-128/256）
- 地址端口解析
- 加密方法验证

---

### 加密测试

加密测试验证密码学模块的正确性和安全性。

#### Aead — AEAD 加密

**文件**: `tests/Aead.cpp`

测试内容：
- AES-128-GCM 加解密
- AES-256-GCM 加解密
- ChaCha20-Poly1305 加解密
- 认证失败检测

```cpp
// Aead 测试示例
void TestAesGcm() {
    TestRunner runner("Aead");
    
    // 生成密钥和 nonce
    std::array<uint8_t, 16> key = random_bytes<16>();
    std::array<uint8_t, 12> nonce = random_bytes<12>();
    
    // 加密
    std::string plaintext = "Hello, World!";
    std::string aad = "additional data";
    
    auto ciphertext = aead::encrypt(key, nonce, plaintext, aad);
    runner.Check(ciphertext.size() == plaintext.size() + 16, "ciphertext has tag");
    
    // 解密
    auto decrypted = aead::decrypt(key, nonce, ciphertext, aad);
    runner.Check(decrypted == plaintext, "decryption matches");
    
    // 认证失败
    ciphertext[0] ^= 1;  // 破坏密文
    auto result = aead::decrypt(key, nonce, ciphertext, aad);
    runner.Check(!result.has_value(), "authentication failure detected");
    
    runner.Summary();
}
```

#### Blake3 — BLAKE3 哈希

**文件**: `tests/Blake3.cpp`

测试内容：
- 基础哈希计算
- 密钥派生（KDF 模式）
- 增量哈希

#### Hkdf — HKDF 密钥派生

**文件**: `tests/Hkdf.cpp`

测试内容：
- Extract 函数
- Expand 函数
- ExpandLabel 函数（TLS 1.3）

#### X25519 — X25519 密钥交换

**文件**: `tests/X25519.cpp`

测试内容：
- 密钥生成
- 公钥派生
- 密钥交换一致性

---

### 多路复用测试

多路复用测试验证 smux 和 yamux 协议的帧处理。

#### Smux — smux 帧处理

**文件**: `tests/Smux.cpp`

测试内容：
- SYN 帧解析
- PSH 帧解析
- FIN 帧解析
- NOP 帧解析
- 帧序列化

```cpp
// Smux 帧格式
struct smux_frame {
    uint8_t version;   // 0x00
    uint8_t type;      // SYN/PSH/FIN/NOP
    uint16_t stream_id;
    uint32_t length;
    // data[length]
};
```

#### Yamux — yamux 帧处理

**文件**: `tests/Yamux.cpp`

测试内容：
- Data 帧解析
- WindowUpdate 帧解析
- Ping 帧解析
- GoAway 帧解析

```cpp
// Yamux 帧格式
struct yamux_frame {
    uint8_t version;     // 0x00
    uint8_t type;        // Data/WindowUpdate/Ping/GoAway
    uint16_t flags;
    uint32_t stream_id;
    uint32_t length;
    // data or window size
};
```

---

### 伪装测试

伪装测试验证 TLS 伪装方案的正确性。

#### Reality — Reality TLS

**文件**: `tests/Reality.cpp`

测试内容：
- Short ID 生成
- 密钥派生
- 握手验证
- 主动探测响应

#### Shadowtls — ShadowTLS

**文件**: `tests/Shadowtls.cpp`

测试内容：
- v2/v3 版本区别
- 密码验证
- 握手流程
- E2E 集成（ShadowTlsE2E.cpp）

---

### DNS 测试

DNS 测试验证 DNS 解析管道各组件。

#### DnsCache — DNS 缓存

**文件**: `tests/DnsCache.cpp`

测试内容：
- 缓存存储和查找
- TTL 管理
- 过期淘汰
- serve-stale 机制

#### DnsPacket — DNS 报文

**文件**: `tests/DnsPacket.cpp`

测试内容：
- 查询报文构建
- 响应报文解析
- IP 提取
- TTL 计算

#### DnsRules — DNS 规则

**文件**: `tests/DnsRules.cpp`

测试内容：
- 地址映射规则
- CNAME 重定向
- 否定应答处理
- 黑名单匹配

---

### 端到端测试

端到端测试验证完整代理链路。

#### E2E — 完整链路

**文件**: `tests/E2E.cpp`

测试场景：
- SOCKS5 TCP 代理全链路
- HTTP 代理全链路
- Trojan 代理全链路
- VLESS 代理全链路

```cpp
// E2E 测试流程
void TestSocks5E2E() {
    TestRunner runner("E2E");
    
    // 1. 启动代理服务器
    ProxyServer server(config);
    server.start();
    
    // 2. 客户端连接
    Socks5Client client("127.0.0.1", 8081);
    runner.Check(client.connect(), "client connected");
    
    // 3. 代理请求
    runner.Check(client.send_request("example.com", 443), "request sent");
    
    // 4. 数据传输
    auto response = client.exchange("GET / HTTP/1.1\r\n\r\n");
    runner.Check(response.contains("200 OK"), "response received");
    
    // 5. 关闭
    client.close();
    server.stop();
    
    runner.Summary();
}
```

---

### 并发测试

并发测试验证多线程和多协程场景。

**文件**: `tests/concurrency/server.cpp`, `tests/concurrency/client.cpp`

运行方式：

```bash
# 终端 1: 启动 server
build_release/tests/concurrency/server.exe

# 终端 2: 运行 client
build_release/tests/concurrency/client.exe
```

测试内容：
- 高并发连接处理
- 协程调度正确性
- 资源竞争处理

---

### 回归测试

回归测试验证历史 Bug 修复的有效性。

**文件**: `tests/Regression.cpp`

包含已知 Bug 的测试案例，确保修复后不再重现：

```cpp
// Regression 测试示例
void TestHmacStackOverflow() {
    TestRunner runner("Regression");
    
    // Issue #123: HMAC 大数据栈溢出
    std::string large_data(100000, 'x');
    auto hmac_result = crypto::hmac_sha256(key, large_data);
    runner.Check(hmac_result.size() == 32, "large HMAC works");
    
    // Issue #456: X25519 密钥解析边界
    // ...
    
    runner.Summary();
}
```

---

## 使用示例

### 编写新测试

添加新测试的步骤：

1. 创建测试文件 `tests/NewFeature.cpp`

```cpp
#include "common/TestRunner.hpp"
#include "prism/new_feature.hpp"

int main() {
    psm::testing::TestRunner runner("NewFeature");
    
    // 测试用例 1
    {
        auto result = new_feature::basic_operation();
        runner.Check(result.success, "basic operation works");
    }
    
    // 测试用例 2
    {
        auto result = new_feature::edge_case();
        runner.Check(result.expected, "edge case handled");
    }
    
    // 测试用例 3 - 错误处理
    {
        auto result = new_feature::error_handling();
        runner.Check(result.error_code != 0, "error detected");
    }
    
    return runner.Summary();
}
```

2. 注册到 `tests/CMakeLists.txt`

```cmake
# 在 forward_add_test 列表中添加
forward_add_test(NewFeature
    SOURCES NewFeature.cpp
    DEPENDS prism_static_library
)
```

3. 构建并运行

```bash
cmake --build build_release --config Release
build_release/tests/NewFeature.exe
```

---

### 扩展 TestRunner

TestRunner 支持扩展自定义检查：

```cpp
// 扩展测试框架示例
class ExtendedTestRunner : public psm::testing::TestRunner {
public:
    // 添加性能检查
    void CheckPerformance(std::chrono::nanoseconds actual,
                          std::chrono::nanoseconds threshold,
                          std::string_view message) {
        bool passed = actual < threshold;
        Check(passed, fmt::format("{}: {}ns < {}ns", 
              message, actual.count(), threshold.count()));
    }
    
    // 添加内存检查
    void CheckMemory(size_t allocated, size_t expected,
                     std::string_view message) {
        bool passed = allocated <= expected;
        Check(passed, fmt::format("{}: {}B <= {}B",
              message, allocated, expected));
    }
};
```

---

### 集成测试套件

组织相关测试为套件：

```cpp
// tests/suite/ProtocolSuite.cpp
int main() {
    int total_result = 0;
    
    // 运行 HTTP 测试
    total_result += run_http_tests();
    
    // 运行 SOCKS5 测试
    total_result += run_socks5_tests();
    
    // 运行 Trojan 测试
    total_result += run_trojan_tests();
    
    return total_result;
}
```

---

### CTest 标签系统

使用 CTest 标签组织测试：

```cmake
# tests/CMakeLists.txt 中添加标签
set_tests_properties(Http PROPERTIES LABELS "protocol")
set_tests_properties(Socks5 PROPERTIES LABELS "protocol")
set_tests_properties(Aead PROPERTIES LABELS "crypto")
set_tests_properties(DnsCache PROPERTIES LABELS "dns")
```

按标签运行：

```bash
# 只运行协议测试
ctest --test-dir build_release -L protocol

# 只运行加密测试
ctest --test-dir build_release -L crypto

# 排除 DNS 测试
ctest --test-dir build_release -E dns
```

---

## 最佳实践

### 测试命名规范

| 类型 | 前缀 | 示例 |
|------|------|------|
| 功能测试 | Test | TestBasicGetRequest |
| 边界测试 | TestEdge | TestEdgeEmptyRequest |
| 错误测试 | TestError | TestErrorInvalidHeader |
| 性能测试 | TestPerf | TestPerfLargePayload |

### 测试组织建议

```cpp
// 好的测试组织
void TestSocks5Handshake() {
    TestRunner runner("Socks5");
    
    // 分组 1: 版本协商
    TestVersionNegotiation(runner);
    
    // 分组 2: 认证
    TestAuthentication(runner);
    
    // 分组 3: CONNECT 命令
    TestConnectCommand(runner);
    
    // 分组 4: UDP ASSOCIATE
    TestUdpAssociate(runner);
    
    return runner.Summary();
}

// 分组函数
void TestVersionNegotiation(TestRunner& runner) {
    runner.Check(..., "version 5 accepted");
    runner.Check(..., "no auth method supported");
}
```

### 测试数据管理

```cpp
// 使用固定测试数据
constexpr auto kTestRequest = 
    "GET http://example.com/ HTTP/1.1\r\n"
    "Host: example.com\r\n\r\n";

// 使用测试数据生成器
std::string generate_large_request(size_t size) {
    return std::string(size, 'x');
}

// 使用边界值
constexpr auto kMaxDatagramSize = 65535;
```

### 断言 vs Check

| 场景 | 使用方式 |
|------|----------|
| 前置条件失败 | assert（程序终止） |
| 测试结果验证 | Check（记录失败） |
| 性能基准验证 | CheckPerformance |

---

## 常见问题

### Q1: 测试失败如何定位问题？

**A**: 
1. 单独运行失败测试，查看详细输出
2. 使用调试器运行测试
3. 检查最近的代码改动
4. 查看测试代码中的具体断言

```bash
# 单独运行查看详情
build_release/tests/Socks5.exe

# 使用调试器
gdb build_release/tests/Socks5.exe
run
```

### Q2: 如何添加依赖外部资源的测试？

**A**: 
使用 Mock 或测试夹具：

```cpp
// Mock DNS 解析器
class MockResolver : public dns::Resolver {
public:
    Task<IPAddress> resolve(std::string hostname) override {
        // 返回固定测试 IP
        co_return IPAddress::parse("192.168.1.1");
    }
};

// 在测试中使用
void TestWithMock() {
    MockResolver resolver;
    // 使用 resolver 而不是真实 DNS
}
```

### Q3: 测试运行很慢怎么办？

**A**: 
使用并行运行或优化测试：

```bash
# 并行运行（4 线程）
ctest --test-dir build_release -j 4

# 只运行修改相关的测试
ctest --test-dir build_release -R Socks5
```

### Q4: 如何测试协程代码？

**A**: 
使用 `net::io_context` 运行协程：

```cpp
void TestCoroutine() {
    TestRunner runner("Coroutine");
    
    net::io_context io;
    
    // 启动协程
    net::co_spawn(io, async_test_task(), net::detached);
    
    // 运行直到完成
    io.run();
    
    runner.Check(result, "async task completed");
    return runner.Summary();
}

Task<void> async_test_task() {
    auto socket = co_await connect(...);
    auto data = co_await socket.read();
    co_await socket.write(data);
    result = true;
}
```

### Q5: E2E 测试如何确保环境准备？

**A**: 
使用测试夹具或脚本：

```cpp
// E2E 测试夹具
class E2ETestFixture {
public:
    E2ETestFixture() {
        // 启动服务器
        server_ = start_proxy_server();
    }
    
    ~E2ETestFixture() {
        // 清理服务器
        server_.stop();
    }
    
private:
    ProxyServer server_;
};

void TestE2E() {
    E2ETestFixture fixture;  // 自动准备和清理
    TestRunner runner("E2E");
    // ...
}
```

---

## 排障指南

### 问题：测试找不到依赖

**症状**: 测试运行时报错找不到库或模块

**排查步骤**:

1. 检查构建是否完整
   ```bash
   cmake --build build_release --config Release
   ```

2. 检查依赖路径
   ```bash
   # 确认静态库存在
   ls build_release/src/libprism.a
   ```

3. 检查环境变量
   ```bash
   # Windows: PATH 是否包含 DLL 目录
   # Linux: LD_LIBRARY_PATH 是否设置
   ```

---

### 问题：测试随机失败

**症状**: 同一测试有时通过有时失败

**排查步骤**:

1. 检查是否有竞态条件
   - 多线程测试是否同步
   - 异步操作是否正确等待

2. 检查是否有时间依赖
   - 超时值是否合理
   - 是否有网络延迟影响

3. 检查是否有资源竞争
   - 多测试共享资源
   - 端口/文件冲突

解决方案：
- 增加同步点
- 使用固定等待而非依赖时间
- 独立资源或随机端口

---

### 问题：CTest 无输出

**症状**: ctest 运行但无任何输出

**排查步骤**:

1. 使用 verbose 模式
   ```bash
   ctest --test-dir build_release -V
   ```

2. 检查测试是否被发现
   ```bash
   ctest --test-dir build_release -N
   ```

3. 检查 CMakeLists.txt 配置
   ```cmake
   enable_testing()
   include(CTest)
   ```

---

### 问题：测试内存泄漏

**症状**: 测试运行后报告内存泄漏

**排查步骤**:

1. 使用内存检测工具
   ```bash
   # Linux
   valgrind build_release/tests/Session.exe
   
   # Windows
   # 使用 Visual Studio 内存诊断
   ```

2. 检查测试代码
   - 是否有未释放的资源
   - 是否有异常导致的跳过释放

3. 检查被测代码
   - 内存池是否正确清理
   - 对象生命周期是否正确管理

---

### 问题：协程测试挂起

**症状**: 测试协程代码时程序挂起不退出

**排查步骤**:

1. 检查 io_context 是否运行
   ```cpp
   io.run();  // 必须调用
   ```

2. 检查协程是否完成
   ```cpp
   // 使用 use_awaitable 或 detached
   net::co_spawn(io, task(), net::use_awaitable);
   // 或
   net::co_spawn(io, task(), net::detached);
   ```

3. 检查是否有死锁
   - 协程等待永远不会发生的事件
   - strand 使用不当

---

## 相关链接

- [[dev/testing/stress|stress]] — 压力测试体系
- [[core/overview|overview]] — 模块概览
- [[dev/building/configuration|configuration]] — 配置参考
- [[core/architecture|architecture]] — 架构设计
