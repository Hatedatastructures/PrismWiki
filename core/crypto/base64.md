---
layer: core
source:
  - I:/code/Prism/include/prism/crypto/base64.hpp
---

# Base64 编解码

Base64 是一种二进制到文本的编码方案，将二进制数据转换为可打印 ASCII 字符。本模块提供轻量级 Base64 编解码实现，用于 HTTP Basic 认证等场景。

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

## 相关文档

- [[core/crypto/aead|aead]] - AEAD 认证加密
- [[core/crypto/hkdf|hkdf]] - HKDF 密钥派生