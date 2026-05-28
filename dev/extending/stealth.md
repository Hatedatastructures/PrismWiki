---
layer: dev
title: 新伪装方案开发
tags: [dev, extending, stealth]
---

# 新伪装方案开发

在 Prism stealth 层添加新的 TLS 伪装方案，需要实现 scheme 接口并注册到 scheme_registry。

## Stealth 层架构

```
probe::detect() → protocol_type (TLS)
       │
       ▼
stealth::scheme_executor::execute():
  按候选顺序尝试 reality → shadowtls → restls → ... → native
  每个 scheme 的 execute() 返回 scheme_result
```

### 关键文件

| 文件 | 聘责 |
|------|------|
| `include/prism/stealth/registry.hpp` | 方案注册表单例 |
| `include/prism/stealth/executor.hpp` | 方案执行器管道 |
| `include/prism/stealth/<scheme>/scheme.hpp` | 方案接口定义 |
| `src/prism/stealth/<scheme>/scheme.cpp` | 方案实现 |

## Scheme 接口

```cpp
// scheme_context: 输入参数
struct scheme_context {
    shared_transmission inbound;      // 客户端传输层
    const config& cfg;                // 配置引用
    const net::any_io_executor& exec; // 执行器
};

// scheme_result: 输出结果
struct scheme_result {
    protocol::protocol_type detected; // 检测到的协议类型
    shared_transmission transport;    // 处理后的传输层
    fault::code error;                // 错误码
};

// scheme 接口
class scheme {
public:
    virtual auto execute(scheme_context ctx) 
        -> net::awaitable<scheme_result> = 0;
    virtual auto is_enabled() const -> bool = 0;
};
```

### 结果判断逻辑

- `detected = protocol_type::tls` → "不是我，继续下一个方案"
- `detected = 其他` → "匹配成功，使用此方案"

## 开发步骤

### 步骤 1：定义方案配置

文件：`include/prism/config.hpp`

```cpp
struct <scheme>_config {
    std::string host;          // 后端服务器
    std::uint16_t port;        // 后端端口
    std::string password;      // 认证密码
    // ... 其他配置
};
```

### 步骤 2：实现 Scheme 类

文件：`include/prism/stealth/<scheme>/scheme.hpp`

```cpp
namespace prism::stealth::<scheme> {

class scheme {
public:
    explicit scheme(const config& cfg);
    
    auto execute(scheme_context ctx) 
        -> net::awaitable<scheme_result>;
    
    auto is_enabled() const -> bool;

private:
    const config& cfg_;
};

} // namespace prism::stealth::<scheme>
```

### 步骤 3：实现 execute() 方法

文件：`src/prism/stealth/<scheme>/scheme.cpp`

核心流程：
1. 解析 ClientHello（复用 `protocol::tls::parse_client_hello()`）
2. 检测方案特征（SNI、extension、特定字段）
3. 连接后端 TLS 服务器
4. 认证验证
5. 返回 scheme_result

```cpp
auto scheme::execute(scheme_context ctx) -> net::awaitable<scheme_result>
{
    scheme_result result;
    
    // 1. 解析 ClientHello
    auto hello = co_await protocol::tls::parse_client_hello(ctx.inbound);
    
    // 2. 检测方案特征
    if (!has_scheme_feature(hello)) {
        result.detected = protocol::protocol_type::tls;
        result.transport = std::move(ctx.inbound);
        co_return result;
    }
    
    // 3. 连接后端
    auto backend = co_await connect_backend(cfg_.host, cfg_.port, ctx.exec);
    
    // 4. 认证验证
    auto auth_result = co_await do_authentication(ctx.inbound, backend);
    
    if (auth_result.success) {
        result.detected = protocol::protocol_type::<your_protocol>;
        result.transport = std::move(auth_result.transport);
    } else {
        result.error = auth_result.error;
        result.transport = std::move(ctx.inbound);
    }
    
    co_return result;
}
```

### 步骤 4：注册方案

文件：`include/prism/stealth/registry.hpp` 或注册函数中

```cpp
inline void register_all_schemes()
{
    scheme_registry::global().register_scheme(
        "reality", []() { return std::make_unique<reality::scheme>(); });
    scheme_registry::global().register_scheme(
        "shadowtls", []() { return std::make_unique<shadowtls::scheme>(); });
    // 新增
    scheme_registry::global().register_scheme(
        "<scheme>", []() { return std::make_unique<<scheme>::scheme>(); });
}
```

### 步骤 5：配置优先级

方案按注册顺序尝试。根据方案特征确定优先级：
- 有 ClientHello 特征的方案优先（如 ECH）
- 无特征需探测的方案后置（如 AnyTLS）
- native TLS 作为 fallback

### 步骤 6：编写测试

文件：`tests/<Scheme>.cpp`

测试用例：
- 特征匹配成功
- 特征不匹配（继续下一个方案）
- 认证成功
- 认证失败
- 后端连接失败

## 现有方案参考

| 方案 | 特点 | ClientHello 特征 |
|------|------|-----------------|
| Reality | TLS 握手伪装 + X25519 | 无（需验证 server random） |
| ShadowTLS | TLS 外层 + 内部协议 | 无（服务端先发） |
| Restls | TLS 流量模拟 | 特定 SNI 或 extension |
| ECH | Encrypted ClientHello | `has_ech_extension` |
| AnyTLS | 密码哈希在应用数据 | 无（fallback 方案） |

## 复用现有代码

- ClientHello 解析: `protocol::tls::parse_client_hello()` (`include/prism/protocol/tls/signal.hpp`)
- TLS 后端连接: `src/prism/stealth/reality/handshake.cpp` 的 TLS 握手逻辑
- 密码哈希: `src/prism/crypto/` 中的哈希函数
- 后端连接竞速: `connect::dial::racer`

## 注意事项

- **避免 dynamic_cast**: 使用 `transmission::raw_socket()` 虚方法获取底层 socket
- **TLS 解析复用**: 不要重复实现 ClientHello 解析，复用 `protocol::tls` 共享层
- **协程安全**: 所有 I/O 使用 `co_await`，`co_spawn` lambda 捕获 `self`
- **配置验证**: 在 `is_enabled()` 中检查配置完整性

## 相关页面

- [[extending/overview]] — 扩展开发概览
- [[extending/protocol]] — 新协议接入指南
- [[extending/module]] — 新模块开发
- [[reality]] — Reality 方案详解
- [[shadowtls]] — ShadowTLS 方案详解