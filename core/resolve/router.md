---
title: "router — 分发层路由器"
layer: core
source: "include/prism/resolve/router.hpp"
module: resolve
tags: [router, resolve, dispatch, routing]
updated: 2026-05-27
---

# router — 分发层路由器

`router` 是分发层的顶层门面，整合 DNS 解析器、反向路由表和连接池，为上层提供统一的路由接口。

## 四种路由模式

| 方法 | 用途 | 流程 |
|------|------|------|
| `async_reverse(host)` | 反向代理 | 查表 → 获取端点 → 连接 |
| `async_direct(ep)` | 直连端点 | 直接连接 |
| `async_forward(host, port)` | 正向代理 | DNS 解析 → racer 竞速 → 连接 |
| `async_datagram(host, port)` | 数据报 | DNS 解析 → 创建 UDP socket |

## 设计决策

### 为什么反向路由表用透明哈希？

每次路由查找用 `string_view`（来自协议解析的零拷贝视图）查 `unordered_map<memory::string, endpoint>`。普通哈希表查找需构造临时 `memory::string`，触发 PMR 分配。透明哈希让 `string_view` 直接查表，热路径零分配。

**后果**: `string_hash` 和 `string_equal` 必须保证跨类型哈希一致性。

### 为什么 IP 字面量有短路优化？

`async_forward` 先检测 host 是否为 IP 地址（`make_address`）。如果是，直接构造 endpoint 连接，跳过 DNS 解析。对 IP 直连场景（如连接内网服务），省去一次 DNS 查询的完整开销（UDP/TCP 报文 + 解析器管道 + 缓存查找）。

**后果**: `make_address` 对域名字符串返回错误码，开销约 50ns，远小于 DNS 查询。

### 为什么 async_reverse 未命中时降级为 async_forward？

反向路由表不包含所有域名。未命中的请求可能是新域名或配置缺失，降级为正向代理（DNS 解析 + 连接）保证连通性，而非直接报错。

**后果**: 降级后有 DNS 解析开销，但避免了"反向表未配置就不可达"的问题。

## 成员变量

| 成员 | 类型 | 说明 |
|------|------|------|
| `pool_` | `connection_pool&` | 共享 TCP 连接池 |
| `dns_` | `unique_ptr<dns::resolver>` | DNS 解析器 |
| `reverse_map_` | `hash_map<endpoint>` | 反向路由表（透明哈希） |
| `executor_` | `any_io_executor` | 执行器（UDP socket） |

## 约束

### router 必须比 pool 长寿

**类型**: 生命周期

**规则**: `router` 持有 `connection_pool` 的裸引用，pool 必须在 router 使用期间存活。

**违反后果**: 悬挂引用，未定义行为。

**源码依据**: `router.hpp` 构造函数参数

### async_forward 的 DNS 结果可能为空

**类型**: 状态前置

**规则**: DNS 解析返回空 endpoints 时，返回 `host_unreachable`。调用方应检查错误码。

**违反后果**: 空端点列表传给 racer 会导致无连接发起，超时后返回错误。

## 引用关系

### 依赖

- [[core/resolve/dns/dns|dns::resolver]]：DNS 解析
- [[core/connect/pool/pool|connection_pool]]：TCP 连接池
- [[core/connect/dial/racer|address_racer]]：Happy Eyeballs 竞速

### 被引用

- [[core/outbound/direct|outbound::direct]]：通过 router 进行直连路由
- [[core/instance/worker/worker|worker]]：每个 worker 持有一个 router
