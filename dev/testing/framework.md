---
layer: dev
source: I:/code/Prism/tests/common/TestRunner.hpp
module: testing
type: reference
tags: [testing, framework, testrunner, cpp]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing/overview]]"
  - "[[dev/testing/writing]]"
  - "[[dev/testing/commands]]"
---

# TestRunner 测试框架

Prism 所有测试共用 `tests/common/TestRunner.hpp` 中的轻量级测试运行框架。

## 设计理念

TestRunner 框架的设计原则：

| 原则 | 说明 |
|------|------|
| 无外部依赖 | 不依赖 Google Test 等第三方框架 |
| 轻量级 | 仅依赖 prism::trace 进行日志输出 |
| 统一接口 | 所有测试使用相同的计数器和日志函数 |
| 简洁 API | Check/LogPass/LogFail/Summary 四个核心方法 |

## 源码结构

```cpp
/**
 * @file TestRunner.hpp
 * @brief 轻量级测试运行框架
 * @details 提供统一的测试计数、断言和结果汇总机制，
 * 供所有单元测试复用，消除重复的 passed/failed 计数器和日志函数。
 * 无外部依赖，仅依赖 prism::trace 进行日志输出。
 */

#pragma once

#include <prism/trace/spdlog.hpp>
#include <format>
#include <string_view>

namespace psm::testing
{
    /**
     * @brief 轻量级测试运行器
     * @details 管理测试通过/失败计数器，提供统一的日志输出和结果汇总。
     * 每个测试可执行文件创建一个实例，通过 tag 参数区分不同测试模块的日志来源。
     */
    class TestRunner
    {
    public:
        /**
         * @brief 构造测试运行器
         * @param tag 日志标签，用于区分不同测试模块（如 "Session", "Crypto" 等）
         */
        explicit TestRunner(const std::string_view tag) noexcept
            : tag_(tag)
        {
        }

        /** @brief 获取通过计数 */
        [[nodiscard]] auto PassedCount() const noexcept -> int
        {
            return passed_;
        }

        /** @brief 获取失败计数 */
        [[nodiscard]] auto FailedCount() const noexcept -> int
        {
            return failed_;
        }

        /**
         * @brief 输出信息级别日志
         * @param msg 日志消息
         */
        auto LogInfo(const std::string_view msg) const -> void
        {
            psm::trace::info("[{}] {}", tag_, msg);
        }

        /**
         * @brief 记录测试通过并递增计数器
         * @param msg 测试名称
         */
        auto LogPass(const std::string_view msg) -> void
        {
            ++passed_;
            psm::trace::info("[{}] PASS: {}", tag_, msg);
        }

        /**
         * @brief 记录测试失败并递增计数器
         * @param msg 失败原因
         */
        auto LogFail(const std::string_view msg) -> void
        {
            ++failed_;
            psm::trace::error("[{}] FAIL: {}", tag_, msg);
        }

        /**
         * @brief 检查条件，通过时记录 pass，失败时记录 fail
         * @param condition 待检查的条件
         * @param message 条件描述
         */
        auto Check(const bool condition, const std::string_view message) -> void
        {
            if (condition)
            {
                LogPass(message);
            }
            else
            {
                LogFail(message);
            }
        }

        /**
         * @brief 输出测试结果汇总并返回退出码
         * @details 打印通过/失败计数，关闭日志系统。
         * @return 0 表示全部通过，1 表示存在失败
         */
        [[nodiscard]] auto Summary() -> int
        {
            psm::trace::info("[{}] Results: {} passed, {} failed", tag_, passed_, failed_);
            psm::trace::shutdown();
            return failed_ > 0 ? 1 : 0;
        }

    private:
        std::string_view tag_;
        int passed_ = 0;
        int failed_ = 0;
    };
} // namespace psm::testing
```

## API 详细说明

### 构造函数

```cpp
explicit TestRunner(const std::string_view tag) noexcept
```

创建测试运行器实例，tag 参数用于区分日志来源。

**示例**：
```cpp
psm::testing::TestRunner runner("Socks5");
// 日志输出: [Socks5] PASS: ...
```

### Check 方法

```cpp
auto Check(const bool condition, const std::string_view message) -> void
```

检查条件并自动记录结果。

**示例**：
```cpp
runner.Check(parsed.method == "GET", "method is GET");
runner.Check(parsed.port == 443, "port parsed correctly");
```

### LogPass 方法

```cpp
auto LogPass(const std::string_view msg) -> void
```

手动记录测试通过。

**示例**：
```cpp
runner.LogPass("version negotiation completed");
```

### LogFail 方法

```cpp
auto LogFail(const std::string_view msg) -> void
```

手动记录测试失败。

**示例**：
```cpp
runner.LogFail("unexpected protocol version");
```

### LogInfo 方法

```cpp
auto LogInfo(const std::string_view msg) const -> void
```

输出信息级别日志，不影响计数器。

**示例**：
```cpp
runner.LogInfo("starting handshake test");
```

### Summary 方法

```cpp
[[nodiscard]] auto Summary() -> int
```

输出测试汇总并返回退出码（0=全部通过，1=存在失败）。

**示例**：
```cpp
// 输出: [Socks5] Results: 15 passed, 0 failed
return runner.Summary();  // 返回 0 或 1
```

### PassedCount / FailedCount

```cpp
[[nodiscard]] auto PassedCount() const noexcept -> int
[[nodiscard]] auto FailedCount() const noexcept -> int
```

获取当前计数器值。

**示例**：
```cpp
if (runner.FailedCount() > 0) {
    runner.LogInfo("some tests failed, check output above");
}
```

## 使用示例

### 基本用法

```cpp
#include "common/TestRunner.hpp"
#include <prism/protocol/socks5.hpp>

int main() {
    psm::testing::TestRunner runner("Socks5");
    
    // 测试版本协商
    runner.Check(socks5::version == 0x05, "version is 5");
    
    // 测试认证方法
    runner.Check(socks5::no_auth == 0x00, "no auth method is 0");
    
    // 测试 UDP 头解析
    std::vector<uint8_t> header = {0x00, 0x00, 0x00, 0x01, 192, 168, 1, 1, 0x00, 80};
    auto parsed = socks5::parse_udp_header(header);
    runner.Check(parsed.address == "192.168.1.1", "address parsed");
    runner.Check(parsed.port == 80, "port parsed");
    
    return runner.Summary();
}
```

### 分组测试

```cpp
int main() {
    psm::testing::TestRunner runner("Http");
    
    // 分组 1: GET 请求
    runner.LogInfo("--- GET Request Tests ---");
    TestGetRequest(runner);
    
    // 分组 2: CONNECT 请求
    runner.LogInfo("--- CONNECT Request Tests ---");
    TestConnectRequest(runner);
    
    // 分组 3: POST 请求
    runner.LogInfo("--- POST Request Tests ---");
    TestPostRequest(runner);
    
    return runner.Summary();
}

void TestGetRequest(psm::testing::TestRunner& runner) {
    std::string request = "GET http://example.com/ HTTP/1.1\r\n"
                          "Host: example.com\r\n\r\n";
    auto parsed = http::parse_request(request);
    runner.Check(parsed.method == "GET", "GET method parsed");
    runner.Check(parsed.url == "http://example.com/", "URL parsed");
    runner.Check(parsed.version == "HTTP/1.1", "version parsed");
}
```

### 错误处理测试

```cpp
int main() {
    psm::testing::TestRunner runner("Crypto");
    
    // 测试加密成功
    auto encrypted = aead::encrypt(key, nonce, plaintext, aad);
    runner.Check(encrypted.has_value(), "encryption succeeded");
    
    // 测试解密失败（篡改密文）
    encrypted.value()[0] ^= 1;
    auto decrypted = aead::decrypt(key, nonce, encrypted.value(), aad);
    runner.Check(!decrypted.has_value(), "authentication failure detected");
    
    return runner.Summary();
}
```

## 测试输出格式

TestRunner 输出遵循统一格式：

```
[Socks5] PASS: version is 5
[Socks5] PASS: no auth method is 0
[Socks5] PASS: address parsed
[Socks5] FAIL: port parsed (expected 80, got 443)
[Socks5] Results: 3 passed, 1 failed
```

日志颜色：
- PASS: info 级别（绿色）
- FAIL: error 级别（红色）
- Results: info 级别

## 与外部框架对比

| 特性 | TestRunner | Google Test |
|------|------------|-------------|
| 外部依赖 | 无 | 需要 gtest 库 |
| 二进制大小 | 小 | 较大 |
| API 复杂度 | 简洁 | 丰富 |
| 断言类型 | 基础 | 多种（EXPECT/ASSERT） |
| 测试发现 | 手动注册 | 自动 |
| 死亡测试 | 无 | 支持 |
| Mock 支持 | 手动 | gmock |

Prism 选择自建 TestRunner 的原因：
- 保持构建系统简洁
- 避免外部依赖引入复杂性
- 满足测试需求即可，无需完整框架

## 相关链接

- [[dev/testing/overview]] — 测试体系概述
- [[dev/testing/writing]] — 测试编写指南
- [[dev/testing/commands]] — 测试命令详解
- [[log]] — 日志系统