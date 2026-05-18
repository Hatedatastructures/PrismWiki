---
title: 连接问题排查指南
source:
  - I:/code/Prism/include/prism/channel/
  - I:/code/Prism/src/prism/channel/
module: debugging
type: reference
tags: [connection, network, timeout, disconnect, troubleshooting]
layer: dev
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/debugging/overview]]"
  - "[[core/channel/overview|channel]]"
  - "[[dev/debugging/tcp|tcp]]"
  - "[[dev/debugging/udp|udp]]"
---

# 连接问题排查指南

连接问题是代理服务中最常见的问题类型，包括连接断开、超时、拒绝等。本文档提供系统化的排查方法。

## 概述

### 连接问题分类

| 问题类型 | 典型症状 | 常见原因 |
|----------|----------|----------|
| 连接超时 | 长时间无响应 | 网络不通、防火墙阻断 |
| 连接拒绝 | 立即返回错误 | 端口未监听、服务未启动 |
| 连接断开 | 传输中断 | 网络波动、对端关闭 |
| 连接重置 | RST 包 | 异常关闭、防火墙干预 |
| 连接泄漏 | 资源耗尽 | 未正确释放、僵尸连接 |

### 连接生命周期

```
+------------------+     +------------------+
|     客户端       |     |     服务端       |
+------------------+     +------------------+
         |                       |
         |------- SYN --------->|  建立连接
         |<------ SYN+ACK ------|
         |------- ACK --------->|
         |                       |
         |==== 数据传输 ========|  正常使用
         |                       |
         |------- FIN --------->|  正常关闭
         |<------ FIN+ACK ------|
         |------- ACK --------->|
         |                       |
         v                       v
    TIME_WAIT                CLOSED
```

问题可能发生在任何阶段：

- SYN 阶段：连接超时、拒绝
- 数据阶段：断开、重置
- FIN 阶段：异常关闭

---

## 详解

### 连接超时排查

#### 症状特征

```
日志示例:
[2026-05-17 10:30:45.123] [prism] [error] Connection timeout: host=example.com, port=443, timeout_ms=300
[2026-05-17 10:30:45.456] [prism] [error] Connect failed: code=timeout
```

#### 排查流程

```
连接超时
    |
    v
+------------------------+
| 1. 检查目标可达性       |
| ping target_host       |
| traceroute target_host |
+------------------------+
    |
    v
+------------------------+
| 2. 检查端口状态         |
| telnet host port       |
| nc -zv host port       |
+------------------------+
    |
    v
+------------------------+
| 3. 检查防火墙规则       |
| iptables -L            |
| firewall-cmd --list    |
+------------------------+
    |
    v
+------------------------+
| 4. 检查超时配置         |
| connect_timeout_ms     |
+------------------------+
```

#### 诊断命令

```bash
# 检查网络连通性
ping example.com

# 检查端口可达性
nc -zv example.com 443
telnet example.com 443

# 路由跟踪
traceroute example.com
mtr example.com

# DNS 解析
dig example.com
nslookup example.com
```

#### 配置调整

```json
{
    "pool": {
        "connect_timeout_ms": 5000  // 增加超时时间
    }
}
```

### 连接断开排查

#### 症状特征

```
日志示例:
[2026-05-17 10:35:00.123] [prism] [warn] Connection closed by peer: session_id=abc123
[2026-05-17 10:35:00.456] [prism] [error] Read failed: code=connection_reset
```

#### 常见原因

| 原因 | 日志特征 | 解决方案 |
|------|----------|----------|
| 对端主动关闭 | Connection closed by peer | 正常行为，检查业务逻辑 |
| 网络中断 | Read failed/Write failed | 检查网络稳定性 |
| 心跳超时 | Keepalive timeout | 增加心跳间隔 |
| 资源限制 | Too many connections | 限制连接数 |
| 防火墙干预 | RST received | 检查防火墙配置 |

#### 排查流程

```
连接断开
    |
    v
+------------------------+
| 1. 分析断开时机         |
| - 连接持续时间         |
| - 断开前操作           |
+------------------------+
    |
    v
+------------------------+
| 2. 检查对端状态         |
| - 服务是否重启         |
| - 是否有资源限制       |
+------------------------+
    |
    v
+------------------------+
| 3. 检查网络状态         |
| - 延迟波动             |
| - 丢包率               |
+------------------------+
    |
    v
+------------------------+
| 4. 检查心跳配置         |
| - keep_alive           |
| - keepalive_interval   |
+------------------------+
```

### 连接泄漏排查

#### 症状特征

```
日志示例:
[2026-05-17 10:40:00.123] [prism] [warn] Pool overflow: endpoint=x, cached=64, max=64
[2026-05-17 10:40:00.456] [prism] [error] Resource exhausted: connections=10000
```

#### 排查流程

```
连接泄漏
    |
    v
+------------------------+
| 1. 检查连接池状态       |
| - 缓存连接数           |
| - 僵尸连接             |
+------------------------+
    |
    v
+------------------------+
| 2. 检查会话生命周期     |
| - 是否正确关闭         |
| - 异常处理是否释放     |
+------------------------+
    |
    v
+------------------------+
| 3. 检查清理机制         |
| - cleanup_interval     |
| - max_idle_seconds     |
+------------------------+
```

#### 配置调整

```json
{
    "pool": {
        "max_cache_per_endpoint": 128,     // 增加缓存容量
        "max_idle_seconds": 60,            // 更快清理空闲连接
        "cleanup_interval_sec": 5          // 更频繁清理
    }
}
```

### 连接拒绝排查

#### 症状特征

```
日志示例:
[2026-05-17 10:45:00.123] [prism] [error] Connection refused: host=127.0.0.1, port=8081
[2026-05-17 10:45:00.456] [prism] [error] Listen failed: code=address_in_use
```

#### 常见原因

| 原因 | 检查方法 | 解决方案 |
|------|----------|----------|
| 服务未启动 | systemctl status prism | 启动服务 |
| 端口被占用 | netstat -tlnp | 释放端口或更换端口 |
| 权限不足 | 运行用户权限 | 使用 root 或授权 |
| 地址绑定失败 | 配置 host/port | 修改配置 |

#### 诊断命令

```bash
# 检查端口监听状态
netstat -tlnp | grep 443
ss -tlnp | grep 443

# 检查服务状态
systemctl status prism

# 检查进程绑定
lsof -i :443
```

---

## 日志分析示例

### 正常连接日志

```
[10:30:00.100] [info] Listener accepting: port=443
[10:30:00.150] [info] New connection: from=192.168.1.100:54321
[10:30:00.200] [debug] Session created: id=abc123
[10:30:00.300] [info] Session started: protocol=trojan
```

### 异常连接日志

```
[10:30:00.100] [info] New connection: from=192.168.1.100:54321
[10:30:05.100] [error] Connection timeout: timeout_ms=5000
[10:30:05.150] [warn] Session cleanup: id=abc123, reason=timeout
```

### 连接池满日志

```
[10:30:00.100] [warn] Pool at capacity: endpoint=example.com:443, size=64
[10:30:00.150] [debug] Waiting for available connection
[10:30:00.200] [info] Connection acquired from pool: reuse=true
```

---

## 使用示例

### 网络连通性测试脚本

```bash
#!/bin/bash
# test_connectivity.sh

HOST=$1
PORT=$2

echo "Testing connectivity to $HOST:$PORT"

# 1. DNS 解析
echo "=== DNS Resolution ==="
dig +short $HOST

# 2. Ping 测试
echo "=== Ping Test ==="
ping -c 3 $HOST

# 3. 端口可达性
echo "=== Port Test ==="
nc -zv -w 5 $HOST $PORT

# 4. 路由跟踪
echo "=== Route Trace ==="
traceroute -m 10 $HOST
```

### 连接池监控脚本

```bash
#!/bin/bash
# monitor_connections.sh

# 查看当前连接数
ss -tn state established '( sport = :443 )' | wc -l

# 查看连接详情
ss -tn state established '( sport = :443 )'

# 查看等待关闭的连接
ss -tn state time-wait '( sport = :443 )' | wc -l
```

---

## 最佳实践

### 连接超时配置建议

| 网络环境 | connect_timeout_ms | 建议 |
|----------|--------------------|------|
| 本地网络 | 300-500 | 短超时，快速失败 |
| 国内远程 | 1000-3000 | 中等超时 |
| 国际网络 | 5000-10000 | 长超时，适应延迟 |

### 心跳配置建议

| 场景 | keep_alive | 建议原因 |
|------|------------|----------|
| 长连接代理 | true | 检测僵尸连接 |
| 短连接 | false | 减少开销 |
| 高延迟网络 | true + 间隔调整 | 防止误判 |

### 连接池配置建议

| 场景 | max_cache | max_idle | 建议 |
|------|-----------|----------|------|
| 高并发 | 128-256 | 60 | 大池，长复用 |
| 低并发 | 32-64 | 30 | 小池，快清理 |
| 内存敏感 | 16-32 | 15 | 减少内存占用 |

---

## 常见问题

### Q1: 连接频繁超时怎么办？

**A**: 
1. 增加超时时间
2. 检查网络质量
3. 使用重试机制

### Q2: 连接池满了怎么办？

**A**: 
1. 增加 max_cache_per_endpoint
2. 减少 max_idle_seconds 加快清理
3. 检查连接泄漏

### Q3: 大量 TIME_WAIT 怎么办？

**A**: 
```bash
# Linux 优化
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=30
```

### Q4: 如何防止僵尸连接？

**A**: 
1. 启用 keep_alive
2. 配置合理的 cleanup_interval
3. 使用心跳检测

---

## 排障指南

### 问题：所有连接都超时

**症状**: 无法建立任何连接

**排查步骤**:

1. 检查服务是否运行
   ```bash
   systemctl status prism
   ```

2. 检查端口监听
   ```bash
   ss -tlnp | grep 443
   ```

3. 检查防火墙
   ```bash
   iptables -L -n
   firewall-cmd --list-all
   ```

4. 检查 DNS 解析
   ```bash
   dig example.com
   ```

---

### 问题：连接随机断开

**症状**: 连接不定时断开

**排查步骤**:

1. 检查网络稳定性
   ```bash
   mtr --report example.com
   ```

2. 检查 NAT 超时（如使用 NAT）
   ```bash
   # 检查 NAT 表规则
   iptables -t nat -L
   ```

3. 检查心跳配置
   ```json
   "pool": {
       "keep_alive": true
   }
   ```

---

## 相关链接

- [[dev/debugging/overview]] — 排障体系概述
- [[channel]] — Channel 连接池模块
- [[dev/debugging/tcp|tcp]] — TCP 协议基础
- [[dev/debugging/udp|udp]] — UDP 协议基础
- [[dev/debugging/log-analysis]] — 日志分析方法