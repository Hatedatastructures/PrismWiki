---
title: "Happy Eyeballs 算法"
category: "network"
type: ref
module: ref
source: "RFC 8305, RFC 6555"
tags: [网络, happy-eyeballs, 连接, ipv6, ipv4, 双栈, 优化]
created: 2026-05-15
updated: 2026-05-17
layer: ref
---

# Happy Eyeballs 算法

**类别**: 网络

## 概述

Happy Eyeballs（快乐眼球）是一种用于优化双栈（IPv4/IPv6）网络连接建立的算法，由 RFC 6555 首次定义，后在 RFC 8305 中进行了更新和细化。该算法的核心思想是同时尝试 IPv6 和 IPv4 连接，使用先成功的连接，从而避免 IPv6 连接失败或缓慢时导致的长时间等待问题。

在 IPv6 过渡期间，许多网络环境存在 IPv6 配置问题。某些系统配置了 IPv6 地址但实际无法连通（例如 ISP 不支持 IPv6，或 IPv6 路由配置错误）。如果应用按照传统方式优先尝试 IPv6，可能需要等待超时才能回退到 IPv4，这会导致严重的连接延迟。Happy Eyeballs 通过"竞速"机制解决了这个问题。

Happy Eyeballs 的名称来源于"keep your eyes happy"的说法——算法让用户的眼睛不必长时间等待页面加载，保持"快乐"的状态。这个名称也暗示了算法的核心理念：以用户体验为中心，不要让用户感知到网络层的复杂性。

Happy Eyeballs 的核心价值：

**减少连接延迟**：当 IPv6 连接缓慢或失败时，不需要等待完全超时，IPv4 连接可以快速接管。典型情况下，IPv4 连接尝试延迟 250ms 启动，相比 TCP 连接超时（通常数十秒），延迟大幅减少。

**平滑 IPv6 过渡**：在 IPv6 普及过程中，网络环境复杂多变。Happy Eyeballs 允许 IPv6 和 IPv4 共存，自动选择工作正常的协议，无需用户手动配置。

**提高可用性**：即使一种协议完全不可用，另一种协议仍可工作。这种冗余提高了整体服务可用性。

**透明性**：算法对应用层透明，应用只需调用标准的连接 API，Happy Eyeballs 在底层自动处理。

**标准兼容**：RFC 8305 是 IETF 标准，主流操作系统和浏览器都已实现。Chrome、Firefox、Safari 等浏览器默认启用 Happy Eyeballs。

### 算法演进

**Happy Eyeballs v1（RFC 6555）**：
最初的 Happy Eyeballs 算法，提出了基本的双栈连接竞速概念。主要特点：
- 同时发起 IPv6 和 IPv4 连接尝试
- IPv4 连接延迟启动
- 使用第一个成功的连接

**Happy Eyeballs v2（RFC 8305）**：
改进版本，解决了 v1 的一些问题：
- 更精确的时间控制（连接尝试延迟 250ms）
- 支持多个地址的排序和优先级
- 改进 DNS 解析与连接建立的协调
- 增加了缓存机制避免重复竞速
- 更好的错误处理和回退策略

### 适用场景

Happy Eyeballs 适用于以下场景：

**双栈网络环境**：系统同时配置了 IPv4 和 IPv6 地址，需要自动选择可用路径。

**IPv6 部署初期**：IPv6 配置可能存在问题，需要平滑过渡。

**移动网络**：网络类型频繁切换（WiFi/蜂窝），连接状态不稳定。

**CDN 和负载均衡**：服务器有多个地址，需要快速选择最优路径。

**代理服务器**：需要快速建立上游连接，减少用户感知延迟。

## 原理详解

### DNS 解析与地址排序

Happy Eyeballs 的第一步是 DNS 解析获取目标地址。RFC 8305 推荐的解析流程：

**异步 DNS 查询**：
同时发送 A（IPv4）和 AAAA（IPv6）查询，不等待一个查询完成再发送另一个。这确保尽快获取所有可用地址。

```
时间轴：
0ms     发送 AAAA 查询
0ms     发送 A 查询
~50ms   收到 AAAA 响应（可能）
~50ms   收到 A 响应（可能）
```

**地址排序**：
DNS 响应可能返回多个地址，需要按照优先级排序。排序规则考虑：
- 地址范围（本地、ULA、全球）
- 地址稳定性（固定、临时）
- 地址选择策略（RFC 6724）

RFC 6724 默认优先级表：
| 优先级 | 地址类型 |
|--------|---------|
| 50 | ::1/128 (IPv6 回环) |
| 40 | ::ffff:0:0/96 (IPv4 映射) |
| 35 | 2002::/16 (6to4) |
| 30 | 2001::/32 (Teredo) |
| 25 | ULA (fc00::/7) |
| 20 | ::/0 (其他 IPv6) |
| 10 | 127.0.0.0/8 (IPv4 回环) |
| 5 | 169.254.0.0/16 (IPv4 链路本地) |
| 1 | 0.0.0.0/8 (IPv4 默认) |

**目标地址列表生成**：
排序后的地址列表交替排列 IPv6 和 IPv4 地址，形成优先级列表：
```
地址列表示例：
1. 2001:db8::1 (IPv6 #1)
2. 192.0.2.1 (IPv4 #1)
3. 2001:db8::2 (IPv6 #2)
4. 192.0.2.2 (IPv4 #2)
```

### 连接竞速流程

RFC 8305 定义的 Happy Eyeballs v2 连接竞速流程：

```
时间轴示意：

0ms      开始 IPv6 #1 连接尝试
         │
250ms    启动 IPv4 #1 连接尝试（如果 IPv6 #1 尚未成功）
         │
         ├──→ IPv6 #1 成功 → 使用 IPv6 #1
         │
         ├──→ IPv4 #1 成功 → 使用 IPv4 #1（取消 IPv6 #1）
         │
         ├──→ IPv6 #1 失败 → 启动 IPv6 #2
         │
         └──→ IPv4 #1 失败 → 启动 IPv4 #2
```

**详细步骤**：

**步骤 1**：获取地址列表
1. 异步发送 A 和 AAAA DNS 查询
2. 收集所有响应地址
3. 按照 RFC 6724 排序
4. 生成交替的地址列表

**步骤 2**：启动首个连接尝试
1. 选择列表中的第一个地址（通常是 IPv6）
2. 发起非阻塞连接尝试
3. 记录启动时间

**步骤 3**：启动次级连接尝试
1. 等待首个连接成功或超时
2. 如果 250ms 内首个连接未成功，启动次级地址连接
3. 次级地址通常是不同地址族的第一个地址

**步骤 4**：处理连接结果
1. 监控所有活跃连接尝试
2. 第一个成功的连接被采用
3. 取消其他进行中的连接尝试
4. 关闭未使用的 socket

**步骤 5**：缓存结果
1. 记录成功连接的地址族
2. 下次连接时优先使用该地址族
3. 缓存有一定过期时间

### 时间参数详解

**Connection Attempt Delay（连接尝试延迟）**：
- 默认值：250ms
- 范围：100-500ms（推荐）
- 作用：首个地址族启动后，延迟多长时间启动次级地址族

延迟选择的考量：
- 过短（<100ms）：可能导致 IPv4 先于 IPv6 成功，降低 IPv6 使用率
- 过长（>500ms）：无法有效减少 IPv6 问题导致的延迟
- 250ms：平衡 IPv6 优先和快速回退

**DNS 查询超时**：
- 默认值：与系统 DNS 超时一致
- 推荐值：2-5 秒
- 作用：DNS 查询失败时的最大等待时间

**连接超时**：
- TCP SYN 超时：通常 3-30 秒
- Happy Eyeballs 不改变 TCP 超时，只是提前启动备用连接

### 状态机实现

Happy Eyeballs 可以用状态机描述：

```
                    +----------+
                    |   IDLE   |
                    +----+-----+
                         |
                    DNS 查询开始
                         |
                         v
                    +----------+
          +-------->| RESOLVING|
          |         +----+-----+
          |              |
          |         DNS 响应到达
          |              |
          |              v
          |         +----------+
          |         | SORTING |
          |         +----+-----+
          |              |
          |         地址排序完成
          |              |
          |              v
          |         +----------+
          |         | RACING   |<----+
          |         +----+-----+    |
          |              |          |
          |     成功或全部失败       |
          |              |          |
          |     +--------+--------+ |
          |     |                 | |
          |     v                 v | 启动新尝试
          | +--------+      +-----------+
          | |SUCCESS |      |   FAILED  |
          | +--------+      +-----------+
          |                         |
          +-------------------------+
                    失败但有更多地址
```

**状态说明**：

| 状态 | 说明 |
|------|------|
| IDLE | 初始状态，等待连接请求 |
| RESOLVING | DNS 解析进行中 |
| SORTING | 地址排序中 |
| RACING | 连接竞速进行中 |
| SUCCESS | 连接成功 |
| FAILED | 所有尝试失败 |

### 多地址处理

当 DNS 返回多个地址时的处理策略：

**地址交错策略**：
```
原始列表：[IPv6#1, IPv6#2, IPv4#1, IPv4#2]
交错后：  [IPv6#1, IPv4#1, IPv6#2, IPv4#2]
```

这样确保：
1. IPv6 地址有优先级（通常排在前面）
2. 每个地址族都有早期机会
3. 不会因为一个地址族的问题而完全阻塞

**连接尝试间隔**：
- 同地址族的下一个地址：前一个失败后立即尝试
- 不同地址族：延迟 250ms 启动

**最大并发尝试**：
- 推荐同时最多 2-3 个连接尝试
- 避免过度占用资源
- 按需启动新尝试

### 缓存策略

RFC 8305 推荐缓存成功的地址族：

**缓存目的**：
1. 避免每次连接都进行竞速
2. 快速使用已知可用的路径
3. 减少资源消耗

**缓存内容**：
- 成功的地址族（IPv4 或 IPv6）
- 成功的具体地址（可选）
- 缓存时间和成功率

**缓存过期**：
- 推荐过期时间：10 分钟
- 过期后重新竞速
- 网络变化时清除缓存

**实现考量**：
```cpp
struct happy_eyeballs_cache {
    std::optional<address_family> preferred_family;
    std::chrono::steady_clock::time_point last_success;
    std::chrono::minutes ttl{10};

    auto is_valid() const -> bool {
        auto now = std::chrono::steady_clock::now();
        return preferred_family.has_value() &&
               (now - last_success) < ttl;
    }

    void update(address_family family) {
        preferred_family = family;
        last_success = std::chrono::steady_clock::now();
    }

    void invalidate() {
        preferred_family.reset();
    }
};
```

### 错误处理

**DNS 解析错误**：
- AAAA 查询失败但 A 查询成功：使用 IPv4 地址
- A 查询失败但 AAAA 查询成功：使用 IPv6 地址
- 两者都失败：返回解析错误

**连接错误**：
- 连接被拒绝（RST）：立即尝试下一个地址
- 连接超时：延迟后尝试下一个地址
- 网络不可达：考虑地址族问题

**快速失败**：
- 某些错误（如 RST）表示目标明确拒绝连接
- 不应该重试同一地址族的其他地址
- 记录错误类型用于判断

### 实现示例

以下是 Happy Eyeballs 的简化实现逻辑：

```cpp
// Happy Eyeballs 连接竞速
class happy_eyeballs_connector {
public:
    auto connect(std::string_view host, uint16_t port)
        -> net::awaitable<tcp::socket> {

        // 1. 检查缓存
        if (cache_.is_valid()) {
            auto result = co_await try_connect_cached(host, port);
            if (result) {
                co_return std::move(*result);
            }
            cache_.invalidate();
        }

        // 2. 异步 DNS 解析
        auto addresses = co_await resolve_addresses(host, port);

        // 3. 地址排序
        auto sorted = sort_addresses(addresses);

        // 4. 连接竞速
        auto result = co_await race_connections(sorted);

        // 5. 更新缓存
        if (result.success) {
            cache_.update(result.family);
        }

        co_return std::move(result.socket);
    }

private:
    auto resolve_addresses(std::string_view host, uint16_t port)
        -> net::awaitable<std::vector<endpoint>> {

        // 同时发起 A 和 AAAA 查询
        auto a_results = resolver_.async_resolve(
            host, std::to_string(port),
            net::ip::tcp::v4(), net::use_awaitable);

        auto aaaa_results = resolver_.async_resolve(
            host, std::to_string(port),
            net::ip::tcp::v6(), net::use_awaitable);

        // 等待任一完成，使用第一个成功的
        // 或等待所有完成
        auto results = co_await net::experimental::make_parallel_group(
            a_results, aaaa_results
        ).async_wait(net::experimental::wait_for_all(), net::use_awaitable);

        // 合并结果
        std::vector<endpoint> all;
        // ... 合并 IPv4 和 IPv6 地址
        co_return all;
    }

    auto race_connections(std::vector<endpoint> const& endpoints)
        -> net::awaitable<connect_result> {

        // 启动首个连接
        auto& first = endpoints[0];
        auto first_result = start_connection(first);

        // 延迟启动次级连接
        net::steady_timer timer{io_context_};
        timer.expires_after(std::chrono::milliseconds(250));
        auto timer_result = timer.async_wait(net::use_awaitable);

        // 等待首个成功或定时器到期
        auto race_result = co_await net::experimental::make_parallel_group(
            first_result, timer_result
        ).async_wait(net::experimental::wait_for_one_success(), net::use_awaitable);

        // 处理结果...
    }

    happy_eyeballs_cache cache_;
    net::io_context& io_context_;
    tcp::resolver resolver_;
};
```

## 在 Prism 中的应用

Prism 作为高性能代理服务器，使用 Happy Eyeballs 优化上游连接建立。

### 连接竞争模块

Prism 在 `channel/eyeball` 模块实现 Happy Eyeballs 算法：

**核心组件**：
- `racer`：连接竞争器，管理多个连接尝试
- `cache`：地址族缓存，记录历史成功结果
- `scheduler`：调度器，控制连接启动时序

### 配置选项

```json
{
    "channel": {
        "eyeball": {
            "enabled": true,
            "connection_attempt_delay_ms": 250,
            "cache_ttl_minutes": 10,
            "max_connection_attempts": 3
        }
    }
}
```

**参数说明**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| enabled | true | 是否启用 Happy Eyeballs |
| connection_attempt_delay_ms | 250 | 次级连接启动延迟 |
| cache_ttl_minutes | 10 | 地址族缓存过期时间 |
| max_connection_attempts | 3 | 最大并发连接尝试数 |

### 实现细节

**地址解析**：
```cpp
// 异步解析 IPv4 和 IPv6 地址
auto resolve_dual_stack(std::string_view host, uint16_t port)
    -> net::awaitable<std::vector<endpoint>> {

    // 并行发起 A 和 AAAA 查询
    auto ipv4_future = resolver.async_resolve(
        host, std::to_string(port),
        tcp::v4(), net::use_awaitable);

    auto ipv6_future = resolver.async_resolve(
        host, std::to_string(port),
        tcp::v6(), net::use_awaitable);

    // 等待两者完成
    auto [ipv4_results, ipv6_results] = co_await net::experimental::make_parallel_group(
        ipv4_future, ipv6_future
    ).async_wait(net::experimental::wait_for_all(), net::use_awaitable);

    // 合并并排序
    co_return sort_and_interleave(ipv4_results, ipv6_results);
}
```

**连接竞速**：
```cpp
// 连接竞速
auto race_connections(std::vector<endpoint> endpoints)
    -> net::awaitable<tcp::socket> {

    if (endpoints.empty()) {
        throw std::runtime_error("No endpoints");
    }

    // 启动首个连接
    auto& first = endpoints.front();
    auto socket = co_await try_connect(first);

    if (socket.has_value()) {
        co_return std::move(*socket);
    }

    // 首个失败，启动竞速
    // ...
}
```

### 性能影响

**优点**：
- 减少连接建立延迟（有问题的 IPv6 环境下减少数秒）
- 提高 IPv6 可用性
- 自动故障转移

**开销**：
- DNS 查询：并行查询无额外延迟
- 额外连接尝试：网络带宽和服务器负载
- 内存：多个 socket 和状态管理

**建议**：
- 启用缓存减少重复竞速
- 控制并发连接数
- 监控连接成功率

### 相关模块链接

| 模块 | 功能 | 文档 |
|------|------|------|
| racer | 连接竞争 | [[core/channel/eyeball/racer|racer]] |
| connection_pool | 连接池 | [[core/channel/connection/pool|connection-pool]] |
| upstream | DNS 上游 | [[core/resolve/dns/upstream|upstream]] |
| router | 路由器 | [[core/resolve/router|router]] |

## 最佳实践

### 时间参数调优

**连接尝试延迟**：
- 默认 250ms 适用于大多数场景
- 高延迟网络可适当增加（300-500ms）
- 低延迟网络可适当减少（150-200ms）

**DNS 超时**：
- 与系统 DNS 超时保持一致
- 通常 2-5 秒
- 考虑 DNS 服务器响应时间

### 缓存策略

**缓存时间**：
- 推荐 5-15 分钟
- 网络稳定时可延长
- 移动网络建议缩短

**缓存失效条件**：
- 缓存时间过期
- 网络接口变化
- 连接失败次数超过阈值

### 错误处理

**快速失败识别**：
- RST（连接被拒绝）：不重试同地址
- Network unreachable：禁用该地址族
- Timeout：尝试下一个地址

**重试策略**：
- 同一地址族最多重试 2 次
- 总重试次数不超过地址数
- 指数退避避免拥塞

### 监控与诊断

**监控指标**：
- IPv4 成功率
- IPv6 成功率
- 平均连接延迟
- 缓存命中率

**日志记录**：
```
[INFO] Happy Eyeballs: IPv6 connected in 120ms
[INFO] Happy Eyeballs: Using cached IPv4 preference
[WARN] Happy Eyeballs: IPv6 failed, falling back to IPv4
```

## 常见问题

### Q1: Happy Eyeballs 会导致 IPv4 优先吗？

**答案**：不会。算法仍然优先尝试 IPv6（通常排在前面的地址），只是提前启动 IPv4 作为备用。如果 IPv6 可用且响应足够快（250ms 内成功），则会使用 IPv6。这保证了"IPv6 优先"的同时提供了回退保障。

### Q2: 为什么选择 250ms 延迟？

**答案**：
- 经验表明大多数 IPv6 连接在 100-200ms 内完成
- 250ms 足够让"健康"的 IPv6 连接成功
- 对于"有问题"的 IPv6，不会等待太久
- 平衡了 IPv6 优先和快速回退

### Q3: 缓存如何处理网络变化？

**答案**：
网络变化（如 WiFi 切换到蜂窝）应该清除缓存，因为新的网络环境可能有不同的 IPv6 可用性。实现应该监听网络事件并相应更新缓存。

### Q4: 如何处理 DNS 解析延迟？

**答案**：
- 并行发起 A 和 AAAA 查询
- 不要等待所有查询完成才开始连接
- 使用第一个到达的地址族开始连接

### Q5: 服务器端需要支持 Happy Eyeballs 吗？

**答案**：
服务器端主要是正确配置双栈网络：
- 确保 IPv6 可达性
- 不要只配置 IPv6 地址但没有路由
- DNS 同时返回 A 和 AAAA 记录

## 排障指南

### IPv6 连接问题诊断

**步骤 1：检查 IPv6 配置**
```bash
# 检查 IPv6 地址
ip -6 addr

# 检查 IPv6 路由
ip -6 route

# 测试 IPv6 连通性
ping6 -c 3 ipv6.google.com
```

**步骤 2：检查 DNS 解析**
```bash
# 查询 AAAA 记录
dig AAAA example.com

# 同时查询 A 和 AAAA
dig ANY example.com
```

**步骤 3：测试双栈连接**
```bash
# 强制 IPv4
curl -4 https://example.com

# 强制 IPv6
curl -6 https://example.com

# 自动选择
curl https://example.com
```

### Happy Eyeballs 调试

**启用详细日志**：
```json
{
    "trace": {
        "log_level": "debug",
        "modules": ["eyeball"]
    }
}
```

**检查缓存状态**：
```cpp
// 调试输出缓存状态
if (cache.is_valid()) {
    spdlog::debug("Happy Eyeballs cache: preferred family = {}",
                  cache.preferred_family() == address_family::ipv6 ? "IPv6" : "IPv4");
} else {
    spdlog::debug("Happy Eyeballs cache: invalid or expired");
}
```

**分析连接时序**：
```cpp
// 记录连接时间线
auto start = steady_clock::now();
auto socket = co_await connector.connect(host, port);
auto elapsed = duration_cast<milliseconds>(steady_clock::now() - start);
spdlog::info("Connection established in {}ms via {}", elapsed.count(),
             socket.local_endpoint().address().is_v6() ? "IPv6" : "IPv4");
```

### 常见问题解决

**问题 1：IPv6 总是被跳过**
- 检查 IPv6 是否正确配置
- 确认 AAAA 记录存在
- 验证 IPv6 路由可达

**问题 2：连接延迟仍然很长**
- 检查是否所有地址都不可达
- 确认 DNS 解析是否缓慢
- 验证网络连接状态

**问题 3：缓存不生效**
- 检查缓存 TTL 配置
- 确认没有网络变化事件
- 验证缓存更新逻辑

## 参考资料

- [RFC 6555 - Happy Eyeballs: Success with Dual-Stack Hosts](https://tools.ietf.org/html/rfc6555)
- [RFC 8305 - Happy Eyeballs Version 2](https://tools.ietf.org/html/rfc8305)
- [RFC 6724 - Default Address Selection](https://tools.ietf.org/html/rfc6724)
- [RFC 8445 - ICE: Interactive Connectivity Establishment](https://tools.ietf.org/html/rfc8445)

## 相关知识

- [[ref/network/tcp|TCP]] — 传输控制协议
- [[ref/network/connection-pool|连接池]] — 连接复用技术
- [[ref/network/nat-traversal|NAT 穿透]] — NAT 穿透技术
- [[core/channel/eyeball/racer|Racer]] — Prism 连接竞争实现
- [[core/resolve/dns/upstream|DNS 上游]] — DNS 解析模块