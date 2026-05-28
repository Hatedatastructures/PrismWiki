---
layer: dev
title: 新模块开发
tags: [dev, extending, module]
---

# 新模块开发

本页介绍 Prism 独立模块的开发方法，包括传输层扩展、DNS 解析器、路由引擎、监控系统等。

## 传输层扩展

### Transmission 接口

```cpp
// include/prism/transport/transmission.hpp
class transmission {
public:
    virtual ~transmission() = default;
    
    // 异步读取
    virtual auto async_read_some(std::span<std::byte> buffer)
        -> net::awaitable<std::size_t> = 0;
    
    // 异步写入
    virtual auto async_write_some(std::span<std::byte const> buffer)
        -> net::awaitable<std::size_t> = 0;
    
    // 关闭连接
    virtual auto close() -> net::awaitable<void> = 0;
    
    // 取消操作
    virtual auto cancel() -> void = 0;
    
    // 获取底层 socket（可选）
    virtual auto raw_socket() noexcept -> tcp::socket* { return nullptr; }
    
    // 判断类型
    virtual auto is_reliable() const noexcept -> bool { return false; }
};
```

### WebSocket 传输示例

文件：`include/prism/transport/websocket.hpp`

```cpp
class websocket final : public transport::transmission {
    shared_transmission inner_;        // 底层 TCP 或 TLS
    memory::string read_buffer_;       // 帧累积缓冲区
    std::size_t frame_offset_{0};     // 当前帧解析偏移
    bool fragmented_{false};           // 分片状态
    
public:
    explicit websocket(shared_transmission inner);
    
    auto async_read_some(std::span<std::byte> buffer)
        -> net::awaitable<std::size_t> override;
    
    auto async_write_some(std::span<std::byte const> buffer)
        -> net::awaitable<std::size_t> override;
    
    auto close() -> net::awaitable<void> override;
    
    // WebSocket 特有方法
    auto send_close_frame(std::uint16_t code) -> net::awaitable<void>;
};
```

### WebSocket 帧格式 (RFC 6455)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-------+---------------+-------------------------------+
|F|  RSV  |  Opcode       |M| Payload len   | Extended len |
+-+-------+---------------+-+-------------+ - - - - - - - +
| Masking Key (if MASK=1) |               |
+ - - - - - - - - - - - - + - - - - - - - + - - - - - - - +
|                    Payload Data                       ...
```

### WebSocket 客户端握手

```cpp
// 发送 HTTP Upgrade
GET /path HTTP/1.1\r\n
Host: host:port\r\n
Upgrade: websocket\r\n
Connection: Upgrade\r\n
Sec-WebSocket-Key: <base64>\r\n
Sec-WebSocket-Version: 13\r\n
\r\n

// 等待响应
HTTP/1.1 101 Switching Protocols\r\n
```

---

## DNS 解析器扩展

### 七阶段 DNS 解析流程

```
1. Cache 查询 → 命中返回
2. Local 解析 → hosts 文件
3. UDP 查询 → 标准查询
4. TCP fallback → TC=1 时
5. DoH 查询 → HTTPS DNS
6. DoT 查询 → TLS DNS
7. 最终结果 → 缓存 + 返回
```

### 添加新 DNS 后端

文件：`include/prism/resolve/dns/upstream.hpp`

```cpp
class dns_upstream {
public:
    virtual auto query(std::span<std::byte const> packet)
        -> net::awaitable<dns_response> = 0;
    
    virtual auto type() const -> upstream_type = 0;
};
```

---

## 路由引擎扩展

### Router 接口

文件：`include/prism/resolve/router.hpp`

```cpp
class router {
public:
    // 反向代理模式（代理客户端）
    auto async_forward(std::string_view host, std::uint16_t port)
        -> net::awaitable<std::pair<fault::code, pooled_connection>>;
    
    // 正向代理模式（代理上游）
    auto async_positive(std::string_view host, std::uint16_t port)
        -> net::awaitable<std::pair<fault::code, pooled_connection>>;
};
```

### 规则引擎

文件：`include/prism/resolve/rules.hpp`

```cpp
// 规则类型
enum class rule_type {
    domain,      // 域名匹配
    domain_suffix, // 域名后缀
    ip_cidr,     // IP CIDR
    geoip,       // GeoIP 国家
    process,     // 进程名
    final,       // 最终规则
};

// 规则匹配
auto match(const request& req) -> rule_result;
```

---

## 监控指标系统

### Metrics 设计

文件：`include/prism/trace/metrics.hpp`

```cpp
namespace prism::trace::metrics {

struct counters {
    std::atomic<std::uint64_t> total_connections{0};
    std::atomic<std::uint64_t> total_bytes_in{0};
    std::atomic<std::uint64_t> total_bytes_out{0};
    std::atomic<std::uint64_t> handshake_errors{0};
    std::atomic<std::uint64_t> dns_lookups{0};
    
    // 按协议分类
    std::atomic<std::uint64_t> http_sessions{0};
    std::atomic<std::uint64_t> socks5_sessions{0};
    std::atomic<std::uint64_t> trojan_sessions{0};
    std::atomic<std::uint64_t> vless_sessions{0};
    std::atomic<std::uint64_t> ss2022_sessions{0};
    
    // 延迟指标
    std::atomic<std::uint64_t> total_handshake_us{0};
    std::atomic<std::uint64_t> handshake_count{0};
};

auto global() -> counters&;

// Prometheus 格式导出
auto prometheus_snapshot() -> std::string;

// JSON 格式导出
auto json_snapshot() -> std::string;

} // namespace prism::trace::metrics
```

### 集成点

| 位置 | 指标更新 |
|------|---------|
| `listener` 接受连接 | `total_connections++` |
| `tunnel()` 读/写 | `total_bytes_in/out += n` |
| dispatch 路由 | `*_sessions++` |
| 握手失败 | `handshake_errors++` |

### 热路径开销

`atomic::fetch_add` 约 1-2 ns（x86 `lock xadd`），在 20 us 连接延迟中占比 <0.01%。

---

## 开发规范

### 协程安全

1. **禁止阻塞调用**: 所有 I/O 使用 `co_await`
2. **生命周期管理**: `co_spawn` lambda 捕获 `self`
3. **PMR 内存**: 热路径使用 `memory::vector<T>`

### PMR 内存使用

```cpp
// 热路径容器
memory::vector<std::byte> buffer{memory::frame_arena()};
memory::string temp{memory::thread_local_pool()};

// 避免使用 std::vector/std::string 在热路径
```

### 错误处理

```cpp
// 热路径: fault::code
if (!result.success) {
    co_return fault::code::connection_failed;
}

// 致命错误: 异常层次结构
throw prism::exception::config_error("invalid port");
```

---

## 相关页面

- [[extending/overview]] — 扩展开发概览
- [[extending/protocol]] — 新协议接入指南
- [[extending/stealth]] — 新伪装方案开发
- [[pmr-memory-pool]] — PMR 内存池详解
- [[modules]] — 模块系统架构