---
layer: dev
title: 并发测试
source: tests/concurrency/
module: testing
type: reference
tags: [testing, concurrency, multi-thread, coroutine]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing/overview]]"
  - "[[dev/testing/commands]]"
  - "[[dev/testing/stress]]"
---

# 并发测试体系

Prism 并发测试验证多线程和多协程环境下的正确性。

## 测试文件结构

```
tests/concurrency/
├── server.cpp          # 测试服务器入口
├── client.cpp          # 测试客户端入口
├── server.hpp          # 服务器声明
├── conversation.hpp    # 会话对话定义
├── handler.hpp         # 协议处理器
└── CMakeLists.txt      # 构建配置
```

## 运行并发测试

并发测试需要同时运行 server 和 client：

### 手动运行

```bash
# 终端 1: 启动 server
build_release/tests/concurrency/server.exe

# 终端 2: 运行 client
build_release/tests/concurrency/client.exe
```

### 自动化脚本

**Windows PowerShell**：

```powershell
# 启动 server（后台）
$server = Start-Process build_release/tests/concurrency/server.exe -PassThru

# 等待 server 启动
Start-Sleep -Seconds 2

# 运行 client
build_release/tests/concurrency/client.exe

# 停止 server
Stop-Process -Id $server.Id
```

**Linux Bash**：

```bash
# 启动 server（后台）
build_release/tests/concurrency/server &
SERVER_PID=$!

# 等待 server 启动
sleep 2

# 运行 client
build_release/tests/concurrency/client

# 停止 server
kill $SERVER_PID
```

## Server — 测试服务器

`server.cpp` 实现测试代理服务器：

```cpp
/**
 * @file server.cpp
 * @brief 并发测试服务器
 * @details 启动代理服务器，等待并发客户端连接
 */

int main() {
    // 初始化
    psm::memory::system::enable_global_pooling();
    psm::trace::init(psm::trace::level::info);
    
    // 加载测试配置
    auto config = load_test_config();
    
    // 创建 worker 线程池
    std::vector<std::unique_ptr<psm::instance::worker>> workers;
    for (size_t i = 0; i < config.worker_count; ++i) {
        workers.emplace_back(std::make_unique<psm::instance::worker>());
    }
    
    // 创建 balancer
    psm::instance::balancer balancer(workers);
    
    // 创建 listener
    psm::instance::listener listener(balancer, config.endpoint);
    
    // 启动
    listener.start();
    
    // 等待信号
    psm::trace::info("Server started on {}", config.endpoint);
    wait_for_shutdown();
    
    return 0;
}
```

### Server 配置

```cpp
struct test_config {
    std::string endpoint = "127.0.0.1:8081";
    size_t worker_count = 4;
    size_t max_connections = 1000;
    std::chrono::seconds timeout{30};
};
```

## Client — 测试客户端

`client.cpp` 实现并发测试客户端：

```cpp
/**
 * @file client.cpp
 * @brief 并发测试客户端
 * @details 启动多个并发连接，验证代理服务器处理能力
 */

int main() {
    psm::testing::TestRunner runner("ConcurrencyClient");
    
    // 测试配置
    const size_t connection_count = 100;
    const size_t requests_per_connection = 100;
    
    // 并发连接测试
    TestConcurrentConnections(runner, connection_count);
    
    // 并发请求测试
    TestConcurrentRequests(runner, connection_count, requests_per_connection);
    
    // 混合协议测试
    TestMixedProtocols(runner);
    
    return runner.Summary();
}
```

### Client 测试场景

```cpp
void TestConcurrentConnections(psm::testing::TestRunner& runner, size_t count) {
    runner.LogInfo("--- Concurrent Connection Test ---");
    
    std::vector<std::thread> threads;
    std::atomic<size_t> success_count{0};
    
    for (size_t i = 0; i < count; ++i) {
        threads.emplace_back([&success_count]() {
            try {
                // 连接服务器
                auto conn = connect_to_server("127.0.0.1", 8081);
                if (conn) {
                    success_count++;
                    conn->close();
                }
            } catch (...) {
                // 连接失败
            }
        });
    }
    
    // 等待所有线程
    for (auto& t : threads) t.join();
    
    runner.Check(success_count == count,
                 fmt::format("{} connections succeeded", success_count.load()));
}
```

## Conversation — 会话对话

`conversation.hpp` 定义测试会话对话：

```cpp
/**
 * @brief 测试会话对话
 * @details 定义客户端-服务器交互序列
 */
struct conversation {
    std::string name;
    std::vector<exchange> exchanges;
};

struct exchange {
    std::vector<uint8_t> request;
    std::vector<uint8_t> expected_response;
    std::chrono::milliseconds timeout{5000};
};

// SOCKS5 认证对话示例
inline conversation socks5_auth_conversation() {
    conversation conv;
    conv.name = "SOCK5 Auth";
    
    // 版本协商
    conv.exchanges.push_back({
        {0x05, 0x01, 0x00},  // 无认证请求
        {0x05, 0x00}         // 无认证响应
    });
    
    // CONNECT 命令
    conv.exchanges.push_back({
        {0x05, 0x01, 0x00, 0x01, 127, 0, 0, 1, 0x00, 80},
        {0x05, 0x00, 0x00, 0x01, 0, 0, 0, 0, 0, 0}  // 成功响应
    });
    
    return conv;
}
```

## Handler — 协议处理器

`handler.hpp` 实现测试处理器：

```cpp
/**
 * @brief 测试处理器
 * @details 处理并发测试中的协议请求
 */
class test_handler {
public:
    void handle_socks5(net::ip::tcp::socket& socket) {
        // 版本协商
        auto version_response = handle_version_negotiation(socket);
        
        // CONNECT 处理
        auto connect_response = handle_connect(socket);
        
        // 数据转发
        handle_data_transfer(socket);
    }
    
    void handle_http(net::ip::tcp::socket& socket) {
        auto request = read_http_request(socket);
        auto response = process_http_request(request);
        write_http_response(socket, response);
    }
};
```

## 测试场景

### 场景 1: 高并发连接

```
连接数: 1000
每连接请求数: 10
协议: SOCKS5
期望: 100% 成功连接
```

测试代码：

```cpp
void TestHighConcurrency(psm::testing::TestRunner& runner) {
    const size_t connections = 1000;
    const size_t requests = 10;
    
    runner.LogInfo(fmt::format("--- {} concurrent connections ---", connections));
    
    std::atomic<size_t> total_success{0};
    std::vector<std::future<void>> futures;
    
    for (size_t i = 0; i < connections; ++i) {
        futures.push_back(std::async(std::launch::async, [&]() {
            auto conn = socks5_connect("127.0.0.1", 8081);
            for (size_t j = 0; j < requests; ++j) {
                auto result = send_request(conn);
                if (result.success) total_success++;
            }
        }));
    }
    
    for (auto& f : futures) f.wait();
    
    runner.Check(total_success == connections * requests,
                 fmt::format("{} requests succeeded", total_success.load()));
}
```

### 场景 2: 混合协议

```
协议: SOCKS5, HTTP, Trojan
连接: 每协议 100
并发: 总 300 连接
```

测试代码：

```cpp
void TestMixedProtocols(psm::testing::TestRunner& runner) {
    runner.LogInfo("--- Mixed Protocol Test ---");
    
    std::atomic<size_t> socks5_success{0};
    std::atomic<size_t> http_success{0};
    std::atomic<size_t> trojan_success{0};
    
    std::vector<std::thread> threads;
    
    // SOCKS5 线程
    for (int i = 0; i < 100; ++i) {
        threads.emplace_back([&]() {
            if (test_socks5()) socks5_success++;
        });
    }
    
    // HTTP 线程
    for (int i = 0; i < 100; ++i) {
        threads.emplace_back([&]() {
            if (test_http()) http_success++;
        });
    }
    
    // Trojan 线程
    for (int i = 0; i < 100; ++i) {
        threads.emplace_back([&]() {
            if (test_trojan()) trojan_success++;
        });
    }
    
    for (auto& t : threads) t.join();
    
    runner.Check(socks5_success == 100, "SOCKS5: 100 connections");
    runner.Check(http_success == 100, "HTTP: 100 connections");
    runner.Check(trojan_success == 100, "Trojan: 100 connections");
}
```

### 场景 3: 持续负载

```
持续时间: 60 秒
并发连接: 500
请求频率: 每连接每秒 10 次
```

测试代码：

```cpp
void TestSustainedLoad(psm::testing::TestRunner& runner) {
    runner.LogInfo("--- Sustained Load Test (60s) ---");
    
    const auto duration = std::chrono::seconds(60);
    const size_t concurrent = 500;
    std::atomic<bool> running{true};
    std::atomic<size_t> request_count{0};
    std::atomic<size_t> error_count{0};
    
    std::vector<std::thread> workers;
    
    for (size_t i = 0; i < concurrent; ++i) {
        workers.emplace_back([&]() {
            auto conn = connect();
            while (running.load()) {
                auto result = send_request(conn);
                if (result.success) request_count++;
                else error_count++;
                std::this_thread::sleep_for(std::chrono::milliseconds(100));
            }
        });
    }
    
    // 运行 60 秒
    std::this_thread::sleep_for(duration);
    running = false;
    
    for (auto& w : workers) w.join();
    
    runner.Check(error_count < request_count * 0.01, 
                 fmt::format("error rate < 1% ({} errors)", error_count.load()));
    runner.LogInfo(fmt::format("Total requests: {}", request_count.load()));
}
```

## 构建配置

`tests/concurrency/CMakeLists.txt`：

```cmake
add_executable(ConcurrencyServer server.cpp)
target_link_libraries(ConcurrencyServer PRIVATE ${PROJECT_NAME}_static_library)

add_executable(ConcurrencyClient client.cpp)
target_link_libraries(ConcurrencyClient PRIVATE ${PROJECT_NAME}_static_library)

if(MINGW)
    target_compile_options(ConcurrencyServer PRIVATE -Wa,-mbig-obj)
    target_compile_options(ConcurrencyClient PRIVATE -Wa,-mbig-obj)
endif()
```

## 输出示例

### Server 输出

```
[Server] Starting on 127.0.0.1:8081
[Server] Worker pool: 4 threads
[Server] Max connections: 1000
[Server] Server ready, waiting for clients...
[Server] Connection 1 accepted
[Server] Connection 2 accepted
...
[Server] Connection 100 accepted
[Server] Shutdown signal received
[Server] Server stopped
```

### Client 输出

```
[ConcurrencyClient] --- Concurrent Connection Test ---
[ConcurrencyClient] PASS: 100 connections succeeded
[ConcurrencyClient] --- Concurrent Request Test ---
[ConcurrencyClient] PASS: 10000 requests succeeded
[ConcurrencyClient] --- Mixed Protocol Test ---
[ConcurrencyClient] PASS: SOCKS5: 100 connections
[ConcurrencyClient] PASS: HTTP: 100 connections
[ConcurrencyClient] PASS: Trojan: 100 connections
[ConcurrencyClient] Results: 4 passed, 0 failed
```

## 验证点

| 验证项 | 期望结果 |
|--------|----------|
| 连接成功率 | >= 99% |
| 请求成功率 | >= 99% |
| 响应时间 | P99 < 500ms |
| 内存泄漏 | 无 |
| 数据完整性 | 100% |

## 与压力测试对比

| 特性 | 并发测试 | 压力测试 |
|------|----------|----------|
| 架构 | Server + Client | 单进程 |
| 目标 | 功能验证 | 稳定性 |
| 时间 | 短时间 | 长时间 |
| 负载 | 模拟负载 | 极限负载 |

## 相关链接

- [[dev/testing/overview]] — 测试体系概述
- [[dev/testing/commands]] — 测试命令详解
- [[dev/testing/stress]] — 压力测试体系
- [[dev/testing/benchmark]] — 基准测试体系
- [[dev/coding/coroutine|coroutine]] — C++23 协程