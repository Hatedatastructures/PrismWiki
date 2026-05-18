---
title: cpp-patterns
created: 2026-05-18
updated: 2026-05-18
layer: skills
tags: [skill, reference]
---

# C++ 文档模式参考

Prism 项目的 C++23 代码中常见的模式和对应的文档写法。

## 模式 1: Boost.Asio 协程

```cpp
auto async_connect(target const& t, executor const& ex)
    -> awaitable<pair<fault::code, shared_transmission>>;
```

**文档写法**：
```markdown
### 函数: async_connect [pure virtual]

**功能**: 建立 TCP 连接到目标

**签名**:
`auto async_connect(target const& t, executor const& ex) -> awaitable<pair<fault::code, shared_transmission>>`

**协程说明**: 无栈协程，co_await 内部 IO 操作
```

**注意**：
- 标注 `[pure virtual]` / `[override]`
- 返回类型是 `awaitable<>` 时说明是协程
- 参数中的 executor 通常不需要详细解释

## 模式 2: CRTP 对象池

```cpp
template <typename T, pool_type Type = pool_type::local>
class pooled_object { ... };
```

**文档写法**：
```markdown
### 类模板: `pooled_object`

CRTP 对象池基类。子类自动获得内存池分配能力。

**签名**: `template <typename T, pool_type Type = pool_type::local> class pooled_object`

**模板参数**:
| 参数 | 约束 | 说明 |
|------|------|------|
| T | 继承 pooled_object 的类 | CRTP 子类类型 |
| Type | pool_type 枚举 | 池类型选择，默认 local |
```

## 模式 3: PMR 容器别名

```cpp
using string = std::pmr::string;
template <typename V> using vector = std::pmr::vector<V>;
```

**文档写法**：
```markdown
### 类型别名

| 别名 | 类型 | 说明 |
|------|------|------|
| `string` | `std::pmr::string` | PMR 字符串 |
| `vector<Value>` | `std::pmr::vector<Value>` | PMR 动态数组 |
```

## 模式 4: glaze 反序列化配置

```cpp
struct config {
    uint32_t timeout_ms = 30000;
    bool enable_ping = true;
    // glaze 反射宏...
};
```

**文档写法**：
```markdown
## 配置项

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| timeout_ms | uint32_t | 30000 | 超时时间 |
| enable_ping | bool | true | 是否启用心跳 |

配置通过 glaze 库从 JSON 反序列化加载。
```

## 模式 5: 装饰器模式（传输层包装）

```cpp
class relay : public transmission {
    shared_transmission inner_;
    ...
};
```

**文档写法**：
```markdown
### 类: relay

Trojan 协议中继器。装饰器模式，包装底层传输层并添加协议处理。

继承自 [[core/channel/transport/transmission]]

**成员变量**:
| 类型 | 名称 | 说明 |
|------|------|------|
| shared_transmission | inner_ | 被包装的底层传输层 |
```

## 模式 6: 操作符重载

```cpp
void* operator new(std::size_t count);
void operator delete(void* ptr, std::size_t count);
```

**文档写法**：
```markdown
## 函数: operator new()

**功能**: 重载单对象 new。小对象从目标池分配，大对象直通系统堆。

**签名**: `void *operator new(std::size_t count)`

**参数**: `count` — 待分配的字节数

**返回值**: `void*` — 分配的内存指针
```

## 模式 7: 枚举类

```cpp
enum class pool_type { global, local };
```

**文档写法**：
```markdown
### 枚举: `pool_type`

内存池类型选择。

| 值 | 说明 |
|----|------|
| `global` | 全局线程安全池 |
| `local` | 线程局部无锁池 |

**签名**: `enum class pool_type`
```

## 模式 8: 回调类型（std::function）

```cpp
using datagram_router_fn = std::function<
    awaitable<pair<code, udp::endpoint>>(string_view, string_view)>;
```

**文档写法**：
```markdown
### 类型别名: `datagram_router_fn`

UDP 数据报路由回调类型。

**签名**: `using datagram_router_fn = std::function<awaitable<pair<code, udp::endpoint>>(string_view, string_view)>`

**参数**: host — 目标主机名；port — 目标端口

**返回值**: `awaitable<pair<code, udp::endpoint>>` — 协程对象
```

## 模式 9: 静态工厂函数

```cpp
static auto make_reliable(shared_transmission inner) -> shared_transmission;
```

**文档写法**：
```markdown
### 函数: make_reliable [static]

**功能**: 将裸传输包装为可靠传输

**签名**: `static auto make_reliable(shared_transmission inner) -> shared_transmission`
```

## 模式 10: noexcept 函数

```cpp
static auto is_ipv6_literal(string_view host) noexcept -> bool;
```

**文档写法**：
```markdown
### 函数: is_ipv6_literal [static, noexcept]

**功能**: 检查目标地址是否为 IPv6 字面量

**签名**: `static auto is_ipv6_literal(string_view host) noexcept -> bool`
```

## 模式 11: Header-only 内联实现

如果模块所有函数都是 header-only（如 loader），在概述中说明：

```markdown
## 概述

Loader 模块负责配置加载。所有函数为 header-only 内联实现，无对应 .cpp 文件。
```

此时 frontmatter 不需要 `> 实现:` 行。

## 模式 12: 协议格式编解码

```cpp
// format.hpp: 编解码辅助
auto encode_request(buffer& buf, request const& req) -> void;
auto decode_request(buffer_view data) -> expected<request, fault::code>;
```

**文档写法**：
```markdown
### 函数: encode_request

**功能**: 将请求结构编码为协议字节流

**签名**: `auto encode_request(buffer& buf, request const& req) -> void`

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| buf | buffer& | 输出缓冲区（追加写入） |
| req | request const& | 请求结构体 |

**返回值**: 无（直接写入 buf）
```
