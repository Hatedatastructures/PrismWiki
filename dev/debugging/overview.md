---
title: 调试与排障体系概述
source:
  - include/prism/trace/
  - src/prism/trace/
  - include/prism/fault/
module: debugging
type: reference
tags: [debugging, troubleshooting, logging, diagnostics, fault]
layer: dev
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/debugging/connection]]"
  - "[[dev/debugging/protocol]]"
  - "[[dev/debugging/memory]]"
  - "[[dev/debugging/performance]]"
  - "[[dev/debugging/tls]]"
  - "[[dev/debugging/log-analysis]]"
  - "[[dev/debugging/common-issues]]"
---

# 调试与排障体系概述

Prism 的调试排障体系建立在完善的日志系统、错误码体系和诊断工具之上。本文档概述排障方法论和工具链。

## 概述

### 排障体系架构

Prism 的调试排障体系分为三层：

```
+--------------------------------------------------+
|                排障体系架构                        |
+--------------------------------------------------+
                       |
         +-------------+-------------+
         |             |             |
    +---------+   +---------+   +---------+
    | 日志层  |   | 错误层  |   | 诊断层 |
    +---------+   +---------+   +---------+
         |             |             |
         v             v             v
    +---------+   +---------+   +---------+
    | trace   |   | fault   |   | 工具   |
    | spdlog  |   | code    |   | wireshark|
    +---------+   +---------+   +---------+
         |             |             |
         +-------------+-------------+
                       |
                       v
              +----------------+
              |  问题定位      |
              +----------------+
```

### 问题分类框架

Prism 的运行问题按来源分类：

| 问题类型 | 来源 | 排障文档 |
|----------|------|----------|
| 连接问题 | 网络层、传输层 | [[dev/debugging/connection]] |
| 协议问题 | 协议解析、编解码 | [[dev/debugging/protocol]] |
| 内存问题 | PMR 池、内存管理 | [[dev/debugging/memory]] |
| 性能问题 | 延迟、吞吐瓶颈 | [[dev/debugging/performance]] |
| TLS 问题 | 加密、证书、握手 | [[dev/debugging/tls]] |

### 排障流程方法论

通用的排障流程：

```
发现问题
    |
    v
+----------------+
| 1. 收集信息    |
| - 日志输出     |
| - 错误码       |
| - 网络抓包     |
+----------------+
    |
    v
+----------------+
| 2. 确定范围    |
| - 问题类型     |
| - 影响范围     |
| - 发生时机     |
+----------------+
    |
    v
+----------------+
| 3. 定位原因    |
| - 代码路径     |
| - 配置参数     |
| - 外部因素     |
+----------------+
    |
    v
+----------------+
| 4. 制定方案    |
| - 修复代码     |
| - 调整配置     |
| - 环境变更     |
+----------------+
    |
    v
+----------------+
| 5. 验证修复    |
| - 测试验证     |
| - 日志确认     |
| - 监控观察     |
+----------------+
```

---

## 详解

### 日志系统基础

Prism 使用 spdlog 实现高性能异步日志。

#### 日志级别

| 级别 | 含义 | 使用场景 |
|------|------|----------|
| trace | 最详细调试 | 开发调试、详细跟踪 |
| debug | 调试信息 | 问题排查、内部状态 |
| info | 常规信息 | 正常运行、关键事件 |
| warn | 警告 | 异常但非致命 |
| error | 错误 | 错误事件、失败操作 |
| critical | 严重错误 | 致命错误、需要立即处理 |

#### 日志格式

Prism 日志默认格式：

```
[2026-05-17 10:30:45.123] [prism] [info] Session started: uuid=xxx
[2026-05-17 10:30:45.456] [prism] [error] Connection failed: code=network_error
```

字段解析：

| 字段 | 含义 |
|------|------|
| 时间戳 | 精确到毫秒的事件时间 |
| 标识 | prism（日志器名称） |
| 级别 | info/error/warn 等 |
| 消息 | 具体事件描述 |

### 错误码体系

Prism 使用 `fault::code` 枚举定义所有错误码。

#### 错误码分类

```
fault::code 枚举（64 值）
    |
    +-- 成功类 (0)
    |   success = 0
    |
    +-- 网络类 (1-15)
    |   network_error, timeout, connection_refused
    |   dns_failure, ssl_error, ...
    |
    +-- 协议类 (16-31)
    |   protocol_error, invalid_request
    |   authentication_failed, ...
    |
    +-- 资源类 (32-47)
    |   out_of_memory, resource_exhausted
    |   pool_overflow, ...
    |
    +-- 内部类 (48-63)
    |   internal_error, unexpected_state
    |   assertion_failed, ...
```

#### 错误码转换为字符串

```cpp
// 使用 fault::to_string()
auto err = fault::code::timeout;
std::string msg = fault::to_string(err);  // "timeout"
```

### 诊断工具链

#### Wireshark 网络分析

Wireshark 是排查网络问题的核心工具：

| 分析场景 | 过滤器 |
|----------|--------|
| TCP 连接 | `tcp.port == 8081` |
| TLS 握手 | `tls.handshake` |
| DNS 查询 | `dns` |
| 代理协议 | `tcp.port == 443 && tcp.payload` |

#### 日志分析脚本

```bash
# 查找错误日志
grep -E "\[error\]|\[critical\]" prism.log

# 查找特定时间段日志
grep "2026-05-17 10:30" prism.log

# 统计错误类型
grep "\[error\]" prism.log | awk -F'code=' '{print $2}' | sort | uniq -c
```

---

## 使用示例

### 基本排障流程

```bash
# 1. 查看当前日志级别配置
cat configuration.json | grep log_level

# 2. 启用 debug 级别日志
# 编辑 configuration.json，设置 log_level: "debug"

# 3. 重启服务
systemctl restart prism

# 4. 触发问题并收集日志
tail -f /var/log/prism/prism.log

# 5. 分析错误
grep -E "\[error\]" prism.log | head -20
```

### 错误码定位

根据错误码定位问题：

```
日志: [error] Connection failed: code=network_error
      |
      v
错误码: network_error (fault::code::network_error)
      |
      v
分类: 网络类错误
      |
      v
参考: [[dev/debugging/connection]]
```

### 抓包分析

```bash
# 抓取 Prism 端口流量
tcpdump -i eth0 port 443 -w prism.pcap

# 用 Wireshark 分析
wireshark prism.pcap
```

---

## 最佳实践

### 日志级别使用建议

| 场景 | 推荐级别 | 原因 |
|------|----------|------|
| 生产环境 | info | 平衡信息量和性能 |
| 问题排查 | debug | 获取详细信息 |
| 开发调试 | trace | 最详细跟踪 |
| 性能敏感 | warn/error | 减少日志开销 |

### 日志保留策略

```json
{
    "trace": {
        "max_size": 134217728,    // 128MB 单文件
        "max_files": 10,          // 10 个轮转文件
        "enable_file": true
    }
}
```

### 错误处理建议

| 场景 | 处理方式 |
|------|----------|
| 网络错误 | 重试 + 超时 |
| 协议错误 | 返回错误响应 |
| 资源错误 | 降级 + 清理 |
| 内部错误 | 记录 + 通知 |

---

## 常见问题

### Q1: 如何提高日志详细度？

**A**: 调整配置文件中的 `log_level`：

```json
{
    "trace": {
        "log_level": "debug"
    }
}
```

### Q2: 日志文件太大怎么办？

**A**: 调整轮转配置：

```json
{
    "trace": {
        "max_size": 67108864,   // 64MB
        "max_files": 8          // 保留 8 个文件
    }
}
```

### Q3: 如何快速定位错误码含义？

**A**: 查阅 [[fault]] 模块文档，或使用代码搜索：

```bash
grep "network_error" include/prism/fault/code.hpp
```

### Q4: 抓包时如何过滤 TLS 加密内容？

**A**: TLS 内容已加密，只能分析握手和元数据。如需分析内容：

- 使用私钥解密（需要配置）
- 分析应用层日志而非抓包

---

## 排障指南

### 问题：日志无输出

**症状**: 运行后无日志文件生成

**排查步骤**:

1. 检查配置文件
   ```bash
   cat configuration.json | grep enable_file
   ```

2. 检查日志目录权限
   ```bash
   ls -la /var/log/prism/
   ```

3. 检查磁盘空间
   ```bash
   df -h
   ```

---

### 问题：日志延迟严重

**症状**: 日志输出延迟数秒

**排查步骤**:

1. 检查队列配置
   ```json
   "queue_size": 8192,  // 增大队列
   "thread_count": 2    // 增加刷盘线程
   ```

2. 检查磁盘 IO
   ```bash
   iostat -x 1
   ```

---

### 深层故障分析

以下文档基于源码深度分析，覆盖各模块的实现边界和深层故障模式：

- [[dev/debugging/deep-dive/protocol-boundaries|协议实现边界与认证深层分析]] — SS2022/Trojan/VLESS/SOCKS5 的深层边界条件
- [[dev/debugging/deep-dive/reality-handshake|Reality 握手深层故障分析]] — 12 步认证 + 5 阶段握手每步的独立失败条件
- [[dev/debugging/deep-dive/shadowtls-limitations|ShadowTLS 深层限制与故障分析]] — 竞态、HMAC 性能退化、无上限检查
- [[dev/debugging/deep-dive/stealth-limitations|伪装方案执行器限制]] — rewind 限制、不可逆点、空壳方案
- [[dev/debugging/deep-dive/recognition-weaknesses|协议识别弱点与误判分析]] — SS2022 误判、VLESS 弱特征、排除法
- [[dev/debugging/deep-dive/system-risks|系统级风险与资源耗尽分析]] — 无优雅关闭、Session 无超时、连接池无上限
- [[dev/debugging/deep-dive/multiplex-boundaries|多路复用边界条件分析]] — pending 超时、资源上限

### 日志参考

- [[dev/debugging/log-reference|完整日志字典]] — 按模块索引的所有日志消息与错误码交叉引用

## 相关链接

- [[dev/debugging/connection]] — 连接问题排查
- [[dev/debugging/protocol]] — 协议问题排查
- [[dev/debugging/memory]] — 内存问题排查
- [[dev/debugging/performance]] — 性能问题排查
- [[dev/debugging/tls]] — TLS 问题排查
- [[dev/debugging/log-analysis]] — 日志分析方法
- [[dev/debugging/common-issues]] — 常见问题汇总
- [[trace]] — Trace 模块详解
- [[fault]] — Fault 错误码体系