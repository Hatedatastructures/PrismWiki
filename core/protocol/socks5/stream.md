---
layer: core
source: I:/code/Prism/include/prism/protocol/socks5/stream.hpp
title: SOCKS5 协议中继器
---

# SOCKS5 协议中继器

实现完整的 SOCKS5 协议（RFC 1928）服务端中继器，提供协程友好的高级 API。

## 源码位置

`I:/code/Prism/include/prism/protocol/socks5/stream.hpp`

## 类定义

```cpp
class relay : public transport::transmission, public std::enable_shared_from_this<relay>
{
public:
    using route_callback = std::function<net::awaitable<...>(...)>;

    explicit relay(shared_transmission next_layer,
                   const config &cfg = {},
                   agent::account::directory *account_dir = nullptr);

    auto handshake() -> net::awaitable<std::pair<fault::code, request>>;
    auto async_write_success(const request &info) -> net::awaitable<fault::code>;
    auto async_write_error(reply_code code) -> net::awaitable<fault::code>;
    auto async_associate(const request &request_info, route_callback route) -> net::awaitable<fault::code>;

    auto async_read_some(...) -> net::awaitable<std::size_t> override;
    auto async_write_some(...) -> net::awaitable<std::size_t> override;
    void close() override;
    void cancel() override;

    auto next_layer() -> transmission&;
    auto release() -> shared_transmission;
    bool is_valid() const;
};
```

## 方法说明

### handshake

执行完整 SOCKS5 握手流程。

**流程阶段**：
1. 方法协商
2. 请求解析
3. 命令检查
4. 响应发送

**返回**：错误码和 `request` 结构（目标地址、端口、命令、传输形态）

### async_write_success

发送成功响应，包含绑定地址和端口。

### async_write_error

发送错误响应，地址字段填充为零。

### async_associate

处理 UDP_ASSOCIATE 请求。

**流程**：
1. 绑定本地 UDP 端口
2. 发送关联成功响应
3. 进入 UDP 数据报转发循环
4. 监听控制面关闭

## 协议流程

### 方法协商阶段

```
Client -> Server: VER(0x05) + NMETHODS + METHODS
Server -> Client: VER(0x05) + METHOD
```

支持认证方法：
- 无认证 (0x00)
- 用户名/密码 (0x02)

### 请求处理阶段

```
Client -> Server: VER + CMD + RSV + ATYP + DST.ADDR + DST.PORT
Server -> Client: VER + REP + RSV + ATYP + BND.ADDR + BND.PORT
```

## 命令处理规则

| 命令 | 要求 | 结果 form |
|---|---|---|
| CONNECT | `enable_tcp = true` | stream |
| UDP_ASSOCIATE | `enable_udp = true` | datagram |
| BIND | `enable_bind = true` | stream |

## UDP_ASSOCIATE 实现

- 空闲超时：客户端在 `udp_idle_timeout` 秒内不发送数据则关闭关联
- 控制面关闭：TCP 控制连接断开时自动关闭 UDP 关联
- 单包转发：每个 UDP 数据报独立路由和转发

## 内部方法

- `negotiated_authentication()` - 方法协商和认证
- `perform_password_auth()` - RFC 1929 认证子协商
- `read_request_header()` - 读取请求头部
- `read_address<N>()` - 读取 IP 地址
- `read_domain_address()` - 读取域名地址
- `associate_loop()` - UDP 主循环
- `relay_single_datagram()` - 单数据报转发

## 设计特点

- **继承 transmission**：可多态使用
- **栈分配缓冲区**：避免热路径堆分配
- **配置只读**：运行时不可修改配置
- **账户租约管理**：认证成功后持有 `account::lease`

## 工厂函数

```cpp
auto make_relay(shared_transmission next_layer,
                const config &cfg = {},
                agent::account::directory *account_dir = nullptr)
    -> shared_relay;
```

## 调用链

```
agent/worker/worker -> protocol/socks5::make_relay -> relay
relay::handshake -> relay::negotiated_authentication -> auth_method
relay::handshake -> relay::read_request_header -> wire::parse_header
relay::handshake -> relay::read_address -> wire::parse_ipv4/parse_ipv6
relay::handshake -> relay::read_domain_address -> wire::parse_domain
relay::async_associate -> relay::associate_loop -> relay::relay_single_datagram
relay::relay_single_datagram -> wire::decode_udp_header
relay::relay_single_datagram -> wire::encode_udp_datagram
```

## 依赖

- [[core/protocol/socks5/constants]] - 协议常量
- [[core/protocol/socks5/config]] - 协议配置
- [[core/protocol/socks5/wire]] - 线级解析
- [[core/protocol/common/address]] - 地址类型
- [[core/protocol/common/form]] - 传输形态
- [[core/transport/transmission]] - 传输层接口
- [[core/instance/account/directory|directory]] - 账户目录
- [[core/crypto/sha224]] - 密码哈希

## 注意事项

- 实例非线程安全，应在同一协程内使用
- 拥有底层传输层的所有权
- 默认仅支持无认证，生产环境需启用认证

## 实现边界

- **UDP ASSOCIATE 仅绑定 IPv4**：`udp::v4()` 硬编码，IPv6 客户端的 UDP ASSOCIATE 静默失败，**完全没有日志**
- **nmethods 最大 256**：处理正确，nmethods=0 时返回"无可用方法"（0xFF）

详见 [[dev/debugging/deep-dive/protocol-boundaries|代理协议实现边界与认证深层分析]]