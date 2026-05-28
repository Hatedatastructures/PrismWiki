---
title: 目标地址结构
created: 2026-05-25
updated: 2026-05-27
layer: core
source: I:/code/Prism/include/prism/protocol/common/target.hpp
tags: [protocol, common, target, host, port]
---

# 目标地址结构

封装解析出的目标主机、端口以及是否使用正向代理标志，是路由决策的关键输入。

> 源码: include/prism/protocol/common/target.hpp | 实现: header-only

## 命名空间

`psm::protocol`

## target

```cpp
struct target
{
    explicit target(memory::resource_pointer mr = memory::current_resource())
        : host(mr), port(mr)
    {
        port.assign("80");
    }

    memory::string host;    // 目标主机名或 IP 地址
    memory::string port;    // 目标端口号，字符串形式
    bool positive{false};   // 是否为正向代理请求
};
```

**路由语义**:
- `positive = true`: 客户端请求使用正向代理
- `positive = false`: 普通请求或反向代理请求

**默认值**: 端口默认为 "80"（HTTP 默认端口）

> **约束**: `host` 和 `port` 字符串可能为空，调用者必须检查有效性。空 host 传递给 dial 层会导致 DNS 解析失败。

**内存管理**: 构造函数接受 `memory::resource_pointer` 参数，成员字符串使用相同内存资源分配，确保与线程局部 PMR 内存池兼容。

## 解析函数

目标地址的解析函数位于 [[core/recognition/target|recognition::target]] 模块中。

## 使用场景

### target 在转发链中的角色

`target` 是协议识别（Recognition）到连接建立（Connect）之间的桥梁数据结构。其流转路径为：

1. **协议 handler 解析请求头**，从客户端数据中提取目标 host/port，填充 `target`
2. **dispatch 层**根据 `target` 决定路由策略（直连、转发、正向代理）
3. **connect::dial** 拿到 `target` 执行 DNS 解析和上游连接
4. **tunnel** 使用 `target` 信息建立双向转发

`positive` 字段标记正向代理意图，影响 dial 层是否跳过本地路由表直连上游。

### 端口默认值 "80" 的由来

HTTP 正向代理是最常见的代理场景。当客户端通过绝对 URI 发起请求（如 `GET http://example.com/path`），若未指定端口，端口字符串保持默认值 "80"。对于 HTTPS CONNECT（端口 443）和 SOCKS5 等协议，端口从协议头中显式解析，覆盖默认值。

## 调用链

- [[core/recognition/target|recognition::resolve]] -> `target`
- [[core/pipeline/primitives|primitives::dial]] -> `target` -> router
