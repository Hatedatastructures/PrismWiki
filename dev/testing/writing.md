---
layer: dev
source: I:/code/Prism/tests/
module: testing
type: guide
tags: [testing, writing, unit-test, cpp]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing/overview]]"
  - "[[dev/testing/framework]]"
  - "[[dev/testing/commands]]"
---

# 测试编写指南

本指南介绍如何在 Prism 项目中编写单元测试、集成测试和端到端测试。

## 测试编写流程

### 1. 创建测试文件

在 `tests/` 目录创建测试文件：

```cpp
// tests/NewFeature.cpp
#include "common/TestRunner.hpp"
#include <prism/new_feature.hpp>

int main() {
    psm::testing::TestRunner runner("NewFeature");
    
    // 测试用例
    TestBasicOperation(runner);
    TestEdgeCases(runner);
    TestErrorHandling(runner);
    
    return runner.Summary();
}
```

### 2. 注册测试

在 `tests/CMakeLists.txt` 中注册：

```cmake
# 在文件末尾添加
forward_add_test(NewFeature NewFeature.cpp)
```

### 3. 构建并运行

```bash
cmake --build build_release --config Release
build_release/tests/NewFeature.exe
```

## 测试命名规范

| 类型 | 前缀 | 示例 |
|------|------|------|
| 功能测试 | Test | TestBasicGetRequest |
| 边界测试 | TestEdge | TestEdgeEmptyRequest |
| 错误测试 | TestError | TestErrorInvalidHeader |
| 性能检查 | TestPerf | TestPerfLargePayload |

**测试函数命名**：使用 PascalCase（大驼峰）

```cpp
void TestBasicOperation(psm::testing::TestRunner& runner);
void TestEdgeCaseEmptyInput(psm::testing::TestRunner& runner);
void TestErrorNullPointer(psm::testing::TestRunner& runner);
```

## 测试组织建议

### 分组测试

将相关测试分组便于维护：

```cpp
int main() {
    psm::testing::TestRunner runner("Socks5");
    
    // 分组测试
    runner.LogInfo("--- Version Negotiation ---");
    TestVersionNegotiation(runner);
    
    runner.LogInfo("--- Authentication ---");
    TestAuthentication(runner);
    
    runner.LogInfo("--- CONNECT Command ---");
    TestConnectCommand(runner);
    
    runner.LogInfo("--- UDP ASSOCIATE ---");
    TestUdpAssociate(runner);
    
    return runner.Summary();
}
```

### 辅助函数

提取通用逻辑为辅助函数：

```cpp
// 生成测试数据
std::string generate_socks5_request(const std::string& addr, uint16_t port) {
    std::string request;
    request.push_back(0x05);  // version
    request.push_back(0x01);  // CONNECT
    request.push_back(0x00);  // reserved
    // ... 构建请求
    return request;
}

// 验证响应
void verify_socks5_response(psm::testing::TestRunner& runner,
                            const std::vector<uint8_t>& response,
                            uint8_t expected_status) {
    runner.Check(response.size() >= 10, "response length valid");
    runner.Check(response[1] == expected_status, "status code correct");
}
```

## 测试数据管理

### 固定测试数据

使用 constexpr 定义固定数据：

```cpp
constexpr auto kTestHttpGet = 
    "GET http://example.com/ HTTP/1.1\r\n"
    "Host: example.com\r\n\r\n";

constexpr std::array<uint8_t, 16> kTestKey = {
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
    0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f
};
```

### 测试数据生成器

对于需要动态生成的数据：

```cpp
std::vector<uint8_t> generate_random_bytes(size_t size) {
    std::vector<uint8_t> bytes(size);
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<uint8_t> dist(0, 255);
    for (auto& b : bytes) b = dist(gen);
    return bytes;
}

std::string generate_large_request(size_t size) {
    return std::string(size, 'x');
}
```

### 边界值测试

覆盖边界条件：

```cpp
void TestEdgeCases(psm::testing::TestRunner& runner) {
    runner.LogInfo("--- Boundary Tests ---");
    
    // 空输入
    auto empty_result = parse(std::string());
    runner.Check(empty_result.error == fault::code::invalid_input, "empty input rejected");
    
    // 最小有效输入
    auto min_result = parse(kMinimalValid);
    runner.Check(min_result.success, "minimal input accepted");
    
    // 最大输入
    auto max_result = parse(kMaxSizeInput);
    runner.Check(max_result.success, "max size accepted");
    
    // 超出边界
    auto overflow_result = parse(kOverflowInput);
    runner.Check(overflow_result.error == fault::code::buffer_overflow, "overflow detected");
}
```

## 协议测试编写

### HTTP 协议测试

```cpp
void TestHttpRequest(psm::testing::TestRunner& runner) {
    // GET 请求
    std::string get_request = "GET http://example.com/ HTTP/1.1\r\n"
                              "Host: example.com\r\n\r\n";
    auto parsed = http::parse_request(get_request);
    runner.Check(parsed.method == "GET", "GET method parsed");
    runner.Check(parsed.url == "http://example.com/", "URL parsed");
    
    // CONNECT 请求
    std::string connect_request = "CONNECT example.com:443 HTTP/1.1\r\n\r\n";
    auto connect_parsed = http::parse_request(connect_request);
    runner.Check(connect_parsed.method == "CONNECT", "CONNECT method parsed");
    runner.Check(connect_parsed.host == "example.com", "host parsed");
    runner.Check(connect_parsed.port == 443, "port parsed");
}
```

### SOCKS5 协议测试

```cpp
void TestSocks5UdpHeader(psm::testing::TestRunner& runner) {
    // 构建 UDP 头
    socks5::udp_header header;
    header.frag = 0;
    header.address = "192.168.1.1";
    header.port = 443;
    
    auto encoded = header.encode();
    runner.Check(encoded.size() == 10, "IPv4 UDP header size is 10");
    runner.Check(encoded[0] == 0x00, "reserved byte 0");
    runner.Check(encoded[1] == 0x00, "reserved byte 1");
    runner.Check(encoded[2] == 0x00, "frag number 0");
    runner.Check(encoded[3] == 0x01, "ATYP IPv4");
    
    // 解析验证
    auto decoded = socks5::parse_udp_header(encoded);
    runner.Check(decoded.frag == 0, "frag matches");
    runner.Check(decoded.address == "192.168.1.1", "address matches");
    runner.Check(decoded.port == 443, "port matches");
}
```

### Trojan 协议测试

```cpp
void TestTrojanRequest(psm::testing::TestRunner& runner) {
    // SHA224 密码
    std::string password = "test_password";
    auto hash = crypto::sha224(password);
    
    // 构建请求
    trojan::request req;
    req.password_hash = hash;
    req.command = trojan::command::connect;
    req.address = "example.com";
    req.port = 443;
    
    auto encoded = req.encode();
    runner.Check(encoded.size() == 56 + 4, "Trojan request size");
    
    // 验证格式
    runner.Check(encoded.substr(0, 56) == hash, "password hash at beginning");
    runner.Check(encoded[56] == 0x01, "CONNECT command");
}
```

## 加密模块测试

### AEAD 加密测试

```cpp
void TestAeadEncryption(psm::testing::TestRunner& runner) {
    // 生成测试密钥
    std::array<uint8_t, 16> key = kTestKey128;
    std::array<uint8_t, 12> nonce = {0};
    
    // 测试数据
    std::string plaintext = "Hello, World!";
    std::string aad = "additional data";
    
    // 加密
    auto ciphertext = aead::aes_gcm_encrypt(key, nonce, plaintext, aad);
    runner.Check(ciphertext.size() == plaintext.size() + 16, "ciphertext has auth tag");
    
    // 解密成功
    auto decrypted = aead::aes_gcm_decrypt(key, nonce, ciphertext, aad);
    runner.Check(decrypted == plaintext, "decryption matches original");
    
    // 认证失败检测
    ciphertext[0] ^= 1;  // 破坏密文
    auto failed = aead::aes_gcm_decrypt(key, nonce, ciphertext, aad);
    runner.Check(!failed.has_value(), "tampered ciphertext rejected");
}
```

### 密钥派生测试

```cpp
void TestHkdf(psm::testing::TestRunner& runner) {
    // HKDF Extract
    std::vector<uint8_t> salt(32, 0x01);
    std::vector<uint8_t> ikm(32, 0x02);
    auto prk = hkdf::extract(salt, ikm);
    runner.Check(prk.size() == 32, "PRK size is 32");
    
    // HKDF Expand
    auto key = hkdf::expand(prk, "key", 16);
    runner.Check(key.size() == 16, "key size correct");
    
    // Expand-Label (TLS 1.3)
    auto derived = hkdf::expand_label(prk, "derived", std::vector<uint8_t>(), 32);
    runner.Check(derived.size() == 32, "derived secret size");
}
```

## 多路复用测试

### smux 帧测试

```cpp
void TestSmuxFrame(psm::testing::TestRunner& runner) {
    // SYN 帧
    smux::frame syn;
    syn.version = 0x00;
    syn.type = smux::frame_type::syn;
    syn.stream_id = 1;
    syn.length = 0;
    
    auto encoded = syn.encode();
    runner.Check(encoded.size() == 8, "SYN frame size is 8");
    runner.Check(encoded[0] == 0x00, "version");
    runner.Check(encoded[1] == 0x01, "SYN type");
    
    // PSH 帧（带数据）
    smux::frame psh;
    psh.type = smux::frame_type::psh;
    psh.stream_id = 1;
    psh.data = std::vector<uint8_t>(100, 'x');
    
    encoded = psh.encode();
    runner.Check(encoded.size() == 8 + 100, "PSH frame with data");
}
```

### yamux 帧测试

```cpp
void TestYamuxFrame(psm::testing::TestRunner& runner) {
    // Data 帧
    yamux::frame data;
    data.version = 0x00;
    data.type = yamux::frame_type::data;
    data.flags = 0;
    data.stream_id = 1;
    data.length = 100;
    
    auto encoded = data.encode();
    runner.Check(encoded.size() == 12, "yamux header size is 12");
    
    // WindowUpdate 帧
    yamux::frame window;
    window.type = yamux::frame_type::window_update;
    window.stream_id = 1;
    window.window_delta = 65536;
    
    encoded = window.encode();
    runner.Check(encoded[8] == 0x00, "WindowUpdate type");
}
```

## 协程测试

### 异步操作测试

```cpp
#include <boost/asio.hpp>

namespace net = boost::asio;

void TestCoroutine(psm::testing::TestRunner& runner) {
    net::io_context io;
    bool completed = false;
    
    // 启动协程
    net::co_spawn(io, [&completed]() -> net::awaitable<void> {
        // 模拟异步操作
        auto executor = co_await net::this_coro::executor;
        net::steady_timer timer(executor);
        timer.expires_after(std::chrono::milliseconds(100));
        co_await timer.async_wait(net::use_awaitable);
        completed = true;
        co_return;
    }, net::detached);
    
    // 运行直到完成
    io.run();
    
    runner.Check(completed, "async task completed");
}
```

### Socket 测试

```cpp
void TestSocketIO(psm::testing::TestRunner& runner) {
    net::io_context io;
    std::string received;
    
    net::co_spawn(io, [&]() -> net::awaitable<void> {
        // 创建本地连接
        net::ip::tcp::acceptor acceptor(io, net::ip::tcp::endpoint(net::ip::tcp::v4(), 0));
        auto server_socket = co_await acceptor.async_accept(net::use_awaitable);
        
        // 客户端连接
        net::ip::tcp::socket client_socket(io);
        co_await client_socket.async_connect(acceptor.local_endpoint(), net::use_awaitable);
        
        // 发送数据
        std::string message = "test";
        co_await net::async_write(server_socket, net::buffer(message), net::use_awaitable);
        
        // 接收数据
        std::array<char, 100> buffer;
        auto bytes = co_await client_socket.async_read_some(net::buffer(buffer), net::use_awaitable);
        received = std::string(buffer.data(), bytes);
        
        co_return;
    }, net::detached);
    
    io.run();
    
    runner.Check(received == "test", "socket I/O works");
}
```

## Mock 和 Stub

### Mock TLS Server

使用 `tests/common/MockTlsServer.hpp` 进行 TLS 测试：

```cpp
#include "common/MockTlsServer.hpp"

void TestTlsHandshake(psm::testing::TestRunner& runner) {
    MockTlsServer server;
    server.start();
    
    // 连接 Mock Server
    auto result = connect_tls(server.endpoint());
    runner.Check(result.success, "TLS handshake succeeded");
    
    server.stop();
}
```

### 自定义 Mock

```cpp
// Mock DNS 解析器
class MockResolver {
public:
    std::optional<ip_address> resolve(const std::string& hostname) {
        if (hostname == "test.example.com") {
            return ip_address::parse("192.168.1.1");
        }
        return std::nullopt;
    }
};

void TestWithMock(psm::testing::TestRunner& runner) {
    MockResolver resolver;
    
    auto result = resolver.resolve("test.example.com");
    runner.Check(result.has_value(), "mock resolver returns value");
    runner.Check(result.value().to_string() == "192.168.1.1", "IP matches");
}
```

## 最佳实践

### 测试独立性

每个测试函数应独立运行，不依赖其他测试的执行顺序或状态：

```cpp
// 错误示例 - 依赖全局状态
int g_counter = 0;

void TestIncrement() {
    g_counter++;  // 修改全局状态
    runner.Check(g_counter == 1, "counter incremented");  // 依赖之前的状态
}

// 正确示例 - 使用局部状态
void TestIncrement(psm::testing::TestRunner& runner) {
    int counter = 0;
    counter++;
    runner.Check(counter == 1, "counter incremented");
}
```

### 避免测试内部逻辑

测试应验证公开接口的行为，而非内部实现：

```cpp
// 错误示例 - 测试内部变量
runner.Check(internal_counter == 5, "internal counter is 5");

// 正确示例 - 测试公开行为
auto result = module.process(input);
runner.Check(result.status == success, "process succeeds");
runner.Check(result.output.size() == expected_size, "output size correct");
```

### 清晰的失败信息

失败信息应明确说明期望和实际值：

```cpp
// 错误示例 - 模糊的信息
runner.Check(result == expected, "test failed");

// 正确示例 - 明确的信息
runner.Check(result == expected, 
              fmt::format("result {} matches expected {}", result, expected));
```

## 相关链接

- [[dev/testing/overview]] — 测试体系概述
- [[dev/testing/framework]] — TestRunner 框架
- [[dev/testing/commands]] — 测试命令详解
- [[dev/testing/benchmark]] — 基准测试
- [[dev/testing/stress]] — 压力测试