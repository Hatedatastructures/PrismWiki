---
layer: core
source: I:/code/Prism/include/prism/protocol/http/parser.hpp
title: HTTP 代理请求解析器
---

# HTTP 代理请求解析器

专为代理场景设计的轻量 HTTP 请求头解析模块，仅提取代理转发所需的最少信息。

## 源码位置

`I:/code/Prism/include/prism/protocol/http/parser.hpp`

## 结构定义

### proxy_request

HTTP 代理请求解析结果，所有字段为 `string_view` 指向原始缓冲区。

```cpp
struct proxy_request
{
    std::string_view method;          // CONNECT、GET、POST
    std::string_view target;          // 绝对 URI 或 host:port
    std::string_view host;            // Host 头字段值
    std::string_view authorization;   // Proxy-Authorization 头字段值
    std::string_view version;         // HTTP/1.1
    std::size_t req_line_end;         // 请求行结束偏移
    std::size_t header_end;           // 头部结束偏移
};
```

### auth_result

HTTP 代理认证结果。

```cpp
struct auth_result
{
    bool authenticated;
    std::string_view error_response;
    agent::account::lease lease;
};
```

## 函数定义

### parse_proxy_request

解析 HTTP 代理请求头。

```cpp
auto parse_proxy_request(std::string_view raw_data, proxy_request &out)
    -> fault::code;
```

**提取字段**：
- 请求方法 (METHOD)
- 目标地址 (TARGET)
- Host 头字段
- Proxy-Authorization 头字段

**请求行格式**：`METHOD TARGET HTTP/version\r\n`

### extract_relative_path

从绝对 URI 中提取相对路径。

```cpp
auto extract_relative_path(std::string_view target) -> std::string_view;
```

**示例**：
- `http://example.com/path?q=1` -> `/path?q=1`
- `/path` -> `/path`（原样返回）

### authenticate_proxy_request

验证 HTTP 代理 Basic 认证。

```cpp
auto authenticate_proxy_request(std::string_view authorization,
                                agent::account::directory &directory)
    -> auth_result;
```

**认证流程**：
1. 验证 Basic 方案前缀
2. Base64 解码凭据
3. 提取密码
4. SHA224 哈希
5. 查询账户目录获取租约

### build_forward_request_line

构建正向代理转发请求行。

```cpp
auto build_forward_request_line(const proxy_request &req,
                                std::pmr::memory_resource *mr)
    -> memory::string;
```

**用途**：将绝对 URI 重写为相对路径，用于正向代理转发。

## 设计目标

- **零堆分配**：所有结果以 `string_view` 指向原始缓冲区
- **最小化解析**：仅提取代理决策所需字段
- **内存高效**：不处理 body 编码和 chunked 传输

## 调用链

```
protocol/http::relay::handshake -> parse_proxy_request -> proxy_request
protocol/http::relay::handshake -> authenticate_proxy_request -> auth_result
protocol/http::relay::forward -> build_forward_request_line -> memory::string
```

## 依赖

- [[core/protocol/http/relay]] - HTTP 代理中继器
- [[core/instance/account/directory]] - 账户目录
- [[core/crypto/base64]] - Base64 解码
- [[core/crypto/sha224]] - 密码哈希

## 注意事项

- `string_view` 依赖原始缓冲区的生命周期
- 不适用于需要完整 HTTP 消息处理的场景