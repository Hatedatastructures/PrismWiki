---
layer: core
source:
  - I:/code/Prism/include/prism/crypto/base64.hpp
tags: [crypto, base64, encoding]
title: Base64 编解码
---

# Base64 编解码

Base64 是一种二进制到文本的编码方案，将二进制数据转换为可打印 ASCII 字符。本模块提供轻量级 Base64 编解码实现，用于 HTTP Basic 认证、SOCKS5 认证等场景。

## 设计决策

### 为什么手写 Base64 而非用 BoringSSL EVP_Base64？

BoringSSL 的 `EVP_EncodeBlock`/`EVP_DecodeBase64` 是较重的 API，引入额外 EVP 上下文开销。Base64 编解码逻辑简单且稳定（RFC 4648），手写 inline 实现可被编译器完全优化，零函数调用开销。同时可精确控制 URL-safe 变体和空白字符处理行为。

**后果**: 模块为 header-only，包含 `<prism/crypto/base64.hpp>` 即可使用，无链接依赖。

### 为什么返回 `memory::string` 而非 `std::string`？

遵循 Prism 的 PMR 内存策略，使用全局池分配器。`memory::string` 是 `std::pmr::string` 的别名，避免在热路径上触发默认堆分配。

**后果**: 返回值可直接参与 PMR 容器操作，但不可直接赋值给 `std::string`（需显式转换）。

### 为什么解码支持 URL-safe 变体？

Prism 的 Base64 主要用于 HTTP Basic Auth 和配置文件，但某些上游组件（如 SOCKS5 或外部 API）可能使用 URL-safe Base64。自动转换 `-`/`_` 到 `+`/`/` 避免调用方需要预处理输入。

**后果**: 解码函数对两种变体透明，但编码函数始终输出标准 Base64（使用 `+`/`/`）。

## 约束

### 解码输入长度必须是 4 的倍数

**类型**: 调用顺序

**规则**: `base64_decode` 的输入字符数（含 padding，不含空白）必须是 4 的倍数

**违反后果**: 返回空字符串

**源码依据**: `base64.hpp:96-99`

### 无效字符检测

**类型**: 调用顺序

**规则**: 输入中的非 Base64 字符（除空白和 padding `=`）导致解码失败

**违反后果**: 返回空字符串

**源码依据**: `base64.hpp:150-153`

### padding 数量上限

**类型**: 资源上限

**规则**: padding 字符 `=` 最多 2 个

**违反后果**: 返回空字符串

**源码依据**: `base64.hpp:90-93`

## 源码位置

- 头文件：`I:/code/Prism/include/prism/crypto/base64.hpp`

## 函数详解

### base64_encode

```cpp
[[nodiscard]] inline auto base64_encode(std::span<const std::uint8_t> input)
    -> std::string;
```

将原始字节数据编码为标准 Base64 字符串。

**参数**：
- `input`：原始字节数据

**返回值**：Base64 编码后的字符串（含 padding）

**编码表**：
```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

**编码规则**：
- 每 3 字节输入编码为 4 字节输出
- 不足 3 字节时用 `=` 填充

### base64_decode

```cpp
[[nodiscard]] inline auto base64_decode(std::string_view input)
    -> std::string;
```

将 Base64 编码字符串解码为原始数据。

**参数**：
- `input`：Base64 编码的字符串

**返回值**：解码后的字符串，解码失败返回空字符串

**特性**：
- 自动忽略空白字符
- 支持 URL-safe 变体（自动转换 `-` 和 `_`）
- 输入长度不是 4 的倍数时返回空字符串
- 无效字符返回空字符串

## Base64 编码原理

### 编码过程

```
输入:  [byte0]  [byte1]  [byte2]
       |________|________|________|
           |        |        |
           v        v        v
       +--------+--------+--------+
       | 6 bits | 6 bits | 6 bits | 6 bits |
       +--------+--------+--------+--------+
           |        |        |        |
           v        v        v        v
       [char0]  [char1]  [char2]  [char3]
```

### Padding 规则

| 输入字节数 | 输出字符数 | Padding |
|-----------|-----------|---------|
| 1 | 2 | `==` |
| 2 | 3 | `=` |
| 3 | 4 | 无 |

### 示例

```
输入: "Man" (0x4D 0x61 0x6E)
二进制: 01001101 01100001 01101110
分组: 010011 010110 000101 101110
索引: 19 22 5 46
输出: "TWFu"

输入: "Ma" (0x4D 0x61)
二进制: 01001101 01100001 00xxxxxx
分组: 010011 010110 000100 xxxxxx
索引: 19 22 4 padding
输出: "TWE="

输入: "M" (0x4D)
二进制: 01001101 00xxxxxx 00xxxxxx
分组: 010011 0100xx xxxxXX xxxxxx
索引: 19 16 padding padding
输出: "TQ=="
```

## URL-safe 变体

标准 Base64 使用 `+` 和 `/`，在 URL 中需要编码。URL-safe 变体替换：

| 标准 | URL-safe |
|------|----------|
| `+` | `-` |
| `/` | `_` |

本模块的 `base64_decode` 自动处理 URL-safe 变体：

```cpp
// 标准格式
auto decoded1 = base64_decode("SGVsbG8rV29ybGQ=");

// URL-safe 格式（自动转换）
auto decoded2 = base64_decode("SGVsbG8tV29ybGQ=");

// 两者解码结果相同
```

## 使用示例

### HTTP Basic 认证

```cpp
// 构造认证头
std::string username = "admin";
std::string password = "secret123";
std::string credentials = username + ":" + password;

// 编码
std::string encoded = base64_encode(
    std::span<const std::uint8_t>(
        reinterpret_cast<const std::uint8_t*>(credentials.data()),
        credentials.size()
    )
);

// 设置 HTTP 头
std::string auth_header = "Authorization: Basic " + encoded;
// "Authorization: Basic YWRtaW46c2VjcmV0MTIz"
```

### 解码 HTTP Basic 认证

```cpp
// 解析认证头
std::string auth_header = "Basic YWRtaW46c2VjcmV0MTIz";
std::string encoded = auth_header.substr(6);  // 跳过 "Basic "

// 解码
std::string decoded = base64_decode(encoded);
// decoded = "admin:secret123"

// 解析用户名和密码
auto pos = decoded.find(':');
std::string username = decoded.substr(0, pos);
std::string password = decoded.substr(pos + 1);
```

### 二进制数据编码

```cpp
// 编码二进制密钥
std::array<std::uint8_t, 32> key = /* ... */;
std::string encoded_key = base64_encode(key);
// 例如: "dGhpcyBpcyBhIDMyIGJ5dGUga2V5IGZvciB0ZXN0aW5n"

// 解码为二进制
std::string decoded = base64_decode(encoded_key);
std::vector<std::uint8_t> binary_key(decoded.begin(), decoded.end());
```

## 错误处理

解码函数在以下情况返回空字符串：

```cpp
// 输入为空
auto r1 = base64_decode("");  // 返回 ""

// 无效字符（非 Base64 字符）
auto r2 = base64_decode("SGVsbG8!V29ybGQ=");  // 返回 ""

// 长度不是 4 的倍数
auto r3 = base64_decode("SGVsbG8");  // 返回 ""

// 过多 padding
auto r4 = base64_decode("SGVsbG8===");  // 返回 ""
```

## 性能特性

| 特性 | 说明 |
|------|------|
| 编码开销 | ~33% 数据膨胀 |
| 解码开销 | ~25% 数据压缩 |
| 内存分配 | 预分配输出大小 |
| 实现 | Header-only inline |

### 内存预分配

```cpp
// 编码预分配
result.reserve(((input.size() + 2) / 3) * 4);

// 解码预分配
result.reserve((valid_count / 4) * 3);
```

## 与其他编码比较

| 编码 | 数据膨胀 | URL 安全 | 用途 |
|------|----------|----------|------|
| Base64 | 33% | 需要 URL-safe 变体 | 通用二进制编码 |
| Base64URL | 33% | 是 | URL 参数 |
| Base32 | 60% | 是 | 不区分大小写场景 |
| Hex | 100% | 是 | 调试、校验和 |

## 调用链

```mermaid
graph TD
    A[base64_encode] --> B[遍历输入字节]
    B --> C[每 3 字节生成 4 字符]
    C --> D[添加 padding]
    
    E[base64_decode] --> F[跳过空白字符]
    F --> G[URL-safe 字符转换]
    G --> H[查表解码]
    H --> I[处理 padding]
```

## 故障场景

### 解码返回空字符串

**触发条件**: 输入不是合法 Base64（长度非 4 倍数、含无效字符、padding 超过 2 个）

**传播路径**: `base64_decode` 返回空字符串 -> 调用方将空字符串作为凭据 -> 认证失败

**外部表现**: HTTP Basic Auth 或 SOCKS5 认证被拒绝

**恢复机制**: 客户端使用正确格式的凭据重试

**日志关键字**: 无直接日志（空字符串返回）

### 编码二进制数据时内存不足

**触发条件**: 输入数据极大导致 `memory::string::reserve` 抛出 `std::bad_alloc`

**传播路径**: 异常向上传播，被 Prism 的异常处理层捕获

**外部表现**: 连接处理中断

**恢复机制**: 内存压力释放后新连接可正常处理

**日志关键字**: 无（异常传播）

### 跨模块契约

| 模块 A | 模块 B | 契约内容 |
|--------|--------|---------|
| [[core/protocol/http/parser\|HTTP parser]] | [[core/crypto/base64\|base64]] | HTTP Basic Auth 解码使用 `base64_decode`，解析 `Authorization: Basic <encoded>` 头 |
| [[core/protocol/socks5/conn\|SOCKS5]] | [[core/crypto/base64\|base64]] | SOCKS5 用户名/密码认证中使用 Base64 编解码 |
| [[core/stealth/reality/handshake\|Reality]] | [[core/crypto/base64\|base64]] | Reality 从配置文件中 Base64 解码私钥字符串 |

## 变更敏感度

### 对外影响

| 变更 | 影响范围 | 影响 |
|------|---------|------|
| 修改返回类型（`memory::string` -> `std::string`） | 全部调用方 | 编译失败或 PMR 策略失效 |
| 修改解码表（`decode_tbl`） | 全部解码调用方 | 解码结果错误 |
| 移除 URL-safe 变体支持 | 使用 URL-safe Base64 的上游组件 | 解码失败 |

### 对内影响

| 上游变更 | 本模块受影响 | 需要检查 |
|---------|------------|---------|
| PMR 内存策略变更 | `memory::string` 定义 | 全部返回值使用方 |
| 新增编解码场景 | 需要支持 Base64URL 编码（无 padding） | 可能需要新增 `base64url_encode` 函数 |

## 相关文档

- [[core/crypto/aead|aead]] - AEAD 认证加密
- [[core/crypto/hkdf|hkdf]] - HKDF 密钥派生