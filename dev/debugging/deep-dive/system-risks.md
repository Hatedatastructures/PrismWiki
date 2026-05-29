---
title: 系统级风险与资源耗尽分析
source:
  - src/prism/instance/worker/worker.cpp
  - src/prism/instance/session/session.cpp
  - src/prism/connect/pool/pool.cpp
  - src/prism/resolve/dns/resolver.cpp
  - src/prism/instance/front/listener.cpp
  - src/prism/connect/tunnel/tunnel.cpp
module: debugging
type: deep-dive
tags: [debugging, system, resource-exhaustion, connection-pool, dns, shutdown, failure-analysis]
layer: dev
created: 2026-05-21
updated: 2026-05-21
related:
  - "[[core/instance/session/session]]"
  - "[[core/multiplex/overview]]"
  - "[[core/resolve/overview]]"
  - "[[dev/debugging/connection]]"
---

# 系统级风险与资源耗尽分析

## 概述

系统级风险是**跨模块传播**的深层问题。单个模块的边界条件缺陷——缺少超时、无上限的资源分配、不可重置的状态——会在运行时通过协程调度、fd 共享、DNS 缓存等机制扩散到整个进程，最终表现为连接拒绝、吞吐骤降或进程僵死。

这类问题通常不会被单元测试覆盖，因为它们需要长时间运行和高并发负载才能触发。排障时往往需要从**症状**（连接超时、日志缺失）反向追溯到**根因**（fd 耗尽、负缓存、阻塞的 accept 循环）。

本文按严重度分级，逐一分析已识别的系统级风险，并给出诊断方法和临时缓解措施。

### 严重度分级

| 级别 | 含义 | 影响范围 | 恢复方式 |
|------|------|----------|----------|
| **P0** | 进程级故障，无法自愈 | 全部用户 | 仅重启 |
| **P1** | 部分功能降级，可短暂持续 | 受影响用户子集 | 等待超时或手动干预 |
| **P2** | 功能缺失或行为偏差，影响有限 | 特定场景 | 可忽略或后续版本修复 |

### 风险速查表

| 症状 | 可能原因 | 严重度 | 影响范围 | 参见章节 |
|------|----------|--------|----------|----------|
| 进程无法正常退出 | 无优雅关闭机制 | P0 | 全部 | [[#无优雅关闭机制]] |
| fd 数量持续增长 | Session 无 idle timeout | P0 | 全部 | [[#Session 无 idle timeout]] |
| 大量 `new connection` 日志 | 连接池无总容量上限 | P1 | 全部 | [[#连接池无总容量上限]] |
| 域名解析连续失败 5 分钟 | DNS 负缓存 TTL 300s | P1 | 特定域名 | [[#DNS 负缓存 TTL 300s]] |
| 新连接接受缓慢 | EMFILE 指数退避不恢复 | P1 | 全部 | [[#EMFILE 指数退避不自动恢复]] |
| accept 后无响应 | 监听器 socket 泄漏 | P1 | 少量连接 | [[#监听器关闭时 socket 泄漏]] |
| HTTP/1.0 管道化异常 | 隧道半关闭语义丢失 | P2 | 特定协议 | [[#隧道半关闭语义丢失]] |
| 无法获取运行时指标 | 统计指标不对外暴露 | P2 | 运维 | [[#统计指标不对外暴露]] |
| 配置修改不生效 | 配置热更新未实现 | P2 | 运维 | [[#配置热更新框架存在但未实现]] |

---

## P0 级风险

### 无优雅关闭机制

#### 机制分析

`main.cpp` 使用 `std::jthread` 管理 worker 线程。每个 worker 内部调用 `ioc_.run()` 驱动 Boost.Asio 事件循环。然而，`ioc_.run()` **永远不返回**——原因是 pool 模块和 metrics 模块各自持有周期性定时器（health check 定时器、metrics 采集定时器），这些定时器不断重新注册，使 `io_context` 始终有未完成的工作。

`std::jthread` 的析构函数会调用 `join()`，等待线程结束。既然 `ioc_.run()` 不返回，`join()` 就永远阻塞。

整个进程**没有信号处理**——既没有 SIGTERM handler，也没有 SIGINT handler。

#### 实际退出方式

当前唯一的退出路径是用户执行 Ctrl+C 或 `kill -9`，进程被操作系统强制终止。此时：

- 所有活跃连接的 TCP RST 被发送到对端
- 正在处理的请求数据丢失
- 连接池中的缓存连接未被正确关闭
- DNS 缓存未清理（影响不大，进程已退出）

#### 排障要点

"无法正常停止"**不是 bug，是缺失的功能**。如果用户报告 `systemctl stop` 超时，问题出在这里。

```
# 现象：systemctl stop prism 超时
# 日志中无 "shutting down" 或 "graceful stop" 相关信息
# 进程在 TimeoutStopSec 后被 SIGKILL

# 典型日志（正常停止尝试）
[Worker 0] ioc_.run() started
... 无任何关闭相关日志 ...
# 进程被 SIGKILL 后日志中断
```

#### 运维缓解

在缺乏优雅关闭机制的情况下，推荐以下 systemd 配置：

```ini
[Service]
# 给进程 10 秒处理时间（实际上做不到优雅关闭）
TimeoutStopSec=10
# 超时后强制终止
KillSignal=SIGKILL
# 不发送 SIGTERM（因为没有 handler，发了也白发）
SendSIGKILL=yes
```

长期方案需要实现信号处理 + io_context 停止 + 连接 drain 的完整关闭流程。

---

### Session 无 idle timeout

#### 机制分析

`session.cpp` 中**没有空闲超时检查**。当一个客户端建立 TCP 连接后不发送任何数据，session 会永远挂在 `recognition::recognize()` 的预读阶段（等待协议识别所需的初始字节）。

更严重的是，Reality 协议的握手过程也**没有超时**。恶意客户端可以在 Reality 握手期间保持连接但不完成握手，使握手协程永久阻塞。

#### 累积效果

```
时间线：
T+0h    : 正常运行，fd 使用 ~200
T+1h    : 100 个恶意连接，fd 使用 ~300
T+4h    : 500 个恶意连接，fd 使用 ~700
T+8h    : 2000 个恶意连接，fd 使用 ~2200
T+12h   : 达到 ulimit -n 上限（通常 1024 或 65535）
         → 新连接被拒绝 (EMFILE)
         → 连接池无法创建新连接
         → DNS resolver 无法创建 UDP socket
         → 整个代理功能瘫痪
```

#### 日志特征

```
# 正常 session 生命周期
[Session] started for 10.0.0.1:54321
[Recognition] protocol identified: vless
[Session] handshake completed
[Session] data relay started

# 恶意 session（无后续日志）
[Session] started for 10.0.0.1:54321
[Session] started for 10.0.0.2:54322
[Session] started for 10.0.0.3:54323
... 大量只有 started 没有后续的 session ...
```

#### 诊断方法

```bash
# 检查进程 fd 数量趋势
watch -n 5 'lsof -p $(pgrep prism) | wc -l'

# 查看系统 fd 使用
cat /proc/sys/fs/file-nr

# 查看进程 fd 上限
cat /proc/$(pgrep prism)/limits | grep "open files"

# 统计已建立但无数据的连接
ss -p | grep prism | grep ESTAB | awk '{print $5}' | sort | uniq -c | sort -rn | head -20
```

#### 缓解措施

```bash
# 临时：增加 fd 上限
prlimit --nofile=65535:65535 --pid $(pgrep prism)

# 临时：使用 iptables 限制单 IP 并发连接
iptables -I INPUT -p tcp --syn -m connlimit --connlimit-above 50 -j DROP

# 长期：在 session 层实现 idle timeout（需要代码修改）
```

---

## P1 级风险

### 连接池无总容量上限

#### 机制分析

`pool.cpp` 中 `max_cache_per_endpoint`（默认值 32）限制了**单个目标地址**的缓存连接数。这是一个合理的单维度限制。

但 `async_acquire()` 在缓存未命中时的行为是**直接创建新连接**，没有全局 fd 上限检查。当并发请求涉及大量不同的目标地址时，每个地址都可能创建最多 32 条缓存连接 + 当前活跃连接。

#### 数学估算

```
假设：
  max_cache_per_endpoint = 32
  不同目标地址数 = 1000（大型代理服务器的典型值）
  每地址活跃连接 = 2

总 fd = 1000 * (32 + 2) = 34,000
加上 session 侧 fd（入站连接）= ~5,000
总计 ≈ 39,000 fd

系统默认 ulimit = 65535 → 余量不足
```

#### 日志特征

```
# 正常：偶尔创建连接
[Pool] new connection to 1.2.3.4:443

# 异常：大量不同目标同时创建连接
[Pool] new connection to 10.0.1.1:443
[Pool] new connection to 10.0.1.2:443
[Pool] new connection to 10.0.1.3:443
[Pool] new connection to 10.0.1.4:443
... 高频出现，目标地址各不相同 ...
```

#### 诊断方法

```bash
# 检查系统 fd 使用
cat /proc/sys/fs/file-nr

# 检查进程 fd 详情
lsof -p $(pgrep prism) | grep TCP | awk '{print $9}' | sort | uniq -c | sort -rn | head -20

# 实时监控
watch -n 1 'cat /proc/$(pgrep prism)/status | grep -i fd'
```

#### 缓解措施

- 降低 `max_cache_per_endpoint`（减少单目标缓存）
- 提高系统 fd 上限：`sysctl -w fs.file-max=2000000`
- 设置进程级限制：`ulimit -n 655350`

---

### DNS 负缓存 TTL 300s

#### 机制分析

`resolver.cpp` 中 `put_negative()` 使用 `negative_ttl`（默认 300 秒 = 5 分钟）将解析失败的域名写入负缓存。

关键问题：即使 DNS 服务已恢复正常，负缓存仍然有效，该域名的**所有查询**在 300 秒内直接返回失败。

#### 与 blacklist 的交互

这是一个特别隐蔽的交互问题。当所有解析出的 IP 被 blacklist 过滤后，结果码为 `fault::code::blocked`，这个结果**也会写入负缓存**。

```
完整故障链：
1. 上游 DNS 临时故障，返回 SERVFAIL
2. resolver 写入负缓存，TTL = 300s
3. 10 秒后 DNS 恢复正常
4. 但负缓存仍然有效（剩余 290 秒）
5. 290 秒内该域名的所有查询直接命中负缓存 → 失败
```

#### 日志序列

```
# T+0: DNS 故障
[Resolver] Stage 4: example.com -> 3 IPs
[Resolver] Stage 5: all IPs blacklisted
[Resolver] negative cache put: example.com, code=blocked, ttl=300

# T+10s: DNS 已恢复，但负缓存命中
[Resolver] Stage 2: negative cache hit for example.com
[Resolver] Stage 2: negative cache hit for example.com
[Resolver] Stage 2: negative cache hit for example.com
... 持续 300 秒 ...

# T+300s: 负缓存过期，恢复正常
[Resolver] Stage 4: example.com -> 3 IPs
[Resolver] Stage 5: selected 1.2.3.4:443
```

#### 诊断方法

```bash
# 观察负缓存命中频率
grep "negative cache hit" /var/log/prism.log | tail -50

# 统计负缓存中的域名数
grep "negative cache put" /var/log/prism.log | awk '{print $NF}' | sort -u | wc -l
```

#### 缓解措施

```yaml
# 减小 negative_ttl（配置文件）
dns:
  negative-ttl: 30    # 从 300s 降至 30s

# 临时清除：重启进程
# 注意：无法热清除负缓存（见 P2：配置热更新未实现）
```

---

### 监听器关闭时 socket 泄漏

#### 机制分析

`listener.cpp` 中 accept 循环使用 `async_accept` 接收新连接。在 accept 返回后、协程将 socket dispatch 给 session 之前，存在一个时间窗口。

当 `ioc_.run()` 被停止时（例如进程退出），accept 循环中的协程被取消。此时已经 `async_accept` 成功返回但尚未 dispatch 的 socket **不会被显式关闭**。

#### 影响评估

- 单次关闭泄漏数量极少（取决于 accept 循环中协程的调度状态）
- 但结合 P0 的"无优雅关闭"，每次强制终止都可能留下少量泄漏
- 对于通过 supervisor 进程频繁重启的场景，泄漏会在 supervisor 层面累积

#### 日志特征

```
# 正常：accept 后立即 dispatch
[Listener] accepted connection from 10.0.0.1:54321
[Session] started for 10.0.0.1:54321

# 泄漏：accept 后进程停止，无后续日志
[Listener] accepted connection from 10.0.0.1:54322
# 进程在此处被停止
```

#### 诊断方法

```bash
# 重启前后对比 fd 数
# 如果泄漏存在，每次重启后 CLOSE_WAIT 状态的连接会增加
ss -p | grep CLOSE_WAIT | grep prism | wc -l
```

---

### EMFILE 指数退避不自动恢复

#### 机制分析

`listener.cpp` 第 104 行使用 `static thread_local` 变量实现 EMFILE 退避：

```cpp
static thread_local std::chrono::milliseconds delay = min_delay;  // 10ms
// EMFILE 时: delay = min(delay * 2, max_delay);  // 最大 5120ms
```

`thread_local static` 的生命周期与线程相同。只要 worker 线程不退出，`delay` 就不会重置。

#### 故障场景

```
1. 正常运行，accept 延迟 ~10ms
2. fd 耗尽风暴（恶意连接攻击或连接泄漏）
3. EMFILE 触发，delay 开始指数增长：10 → 20 → 40 → ... → 5120ms
4. fd 被释放（攻击停止或泄漏被清理）
5. 但 delay 仍然是 5120ms
6. 新连接的接受间隔固定为 5.12 秒
7. 系统完全有能力处理新连接，但被退避延迟人为限制
```

#### 日志特征

```
# 正常
[Listener] accepted connection from 10.0.0.1:54321
[Listener] accepted connection from 10.0.0.1:54322
# 间隔 < 10ms

# EMFILE 期间
[Listener] EMFILE, delaying accept by 20ms
[Listener] EMFILE, delaying accept by 40ms
[Listener] EMFILE, delaying accept by 80ms
...
[Listener] EMFILE, delaying accept by 5120ms

# fd 已释放但延迟未恢复
[Listener] accepted connection from 10.0.0.1:55001
# 5.12 秒后
[Listener] accepted connection from 10.0.0.1:55002
# 5.12 秒后
[Listener] accepted connection from 10.0.0.1:55003
```

#### 缓解措施

```bash
# 临时：重启进程（thread_local 状态随线程销毁而重置）
systemctl restart prism

# 根本：代码层面需要在 fd 释放后重置 delay
# 例如：在 accept 成功后将 delay 重置为 min_delay
```

---

## P2 级风险

### 隧道半关闭语义丢失

#### 机制分析

`tunnel.cpp` 第 84 行使用 `awaitable_operators::||`（并行等待任一完成）：

```cpp
auto result = co_await (
    relay(client, remote, "upstream") ||
    relay(remote, client, "downstream")
);
```

任一方向的 `relay` 完成（通常是读到 EOF），整个 `||` 表达式立即返回。另一个方向的 `relay` 被取消——它不会 drain 剩余数据，而是直接丢弃。

#### 影响范围

| 场景 | 影响 | 严重度 |
|------|------|--------|
| HTTPS 代理（绝大多数场景） | 无影响，TLS 层已处理边界 | 无 |
| HTTP/1.1 Connection: close | 无影响，完整请求/响应已传输 | 无 |
| HTTP/1.0 管道化 | 可能丢失最后一个响应 | 低 |
| WebSocket | 正常关闭帧可能丢失 | 低 |
| 自定义 TCP 协议依赖半关闭 | 数据可能被截断 | 中 |

对大多数代理场景可接受。仅在极少数依赖 TCP 半关闭语义的应用中会出现问题。

---

### 统计指标不对外暴露

#### 机制分析

`state.cpp` 收集了丰富的运行时指标：

| 指标名 | 含义 | 诊断价值 |
|--------|------|----------|
| `active_sessions` | 当前活跃 session 数 | 监控连接泄漏 |
| `pending_handoffs` | 等待移交的连接数 | 监控 accept 压力 |
| `event_loop_lag_us` | 事件循环延迟（微秒） | 监控协程调度延迟 |

但 `worker::load_snapshot()` 仅供 `balancer` 模块内部使用，没有发现 HTTP endpoint、Prometheus metrics endpoint 或其他外部暴露机制。

#### 影响

生产环境缺乏可观测性：
- 无法通过监控系统发现 fd 趋势异常
- 无法设置告警规则（如 active_sessions > 1000）
- 排障时只能依赖日志和系统工具

#### 临时替代方案

```bash
# 从日志中提取近似指标
grep -c "Session.*started" /var/log/prism.log    # 近似 active_sessions
grep "event_loop_lag" /var/log/prism.log | tail -1  # 最近的延迟值

# 使用 systemtap 或 bpftrace 跟踪（高级）
# bpftrace -e 'uprobe:/path/to/prism:*snapshot* { printf("%s\n", str(arg0)); } }'
```

---

### 配置热更新框架存在但未实现

#### 机制分析

`context.hpp` 提供了 `swap_config()` 方法，通过原子交换 `shared_ptr<config>` 实现无锁配置切换。这是一个设计良好的接口。

但搜索整个代码库，**没有任何地方调用** `swap_config()`。

此外，DNS 规则引擎没有清除或重建方法——它在启动时一次性构建，之后无法修改。

#### 现状

| 组件 | 运行时可修改 | 说明 |
|------|-------------|------|
| config 全局配置 | 框架存在，未接入 | `swap_config()` 已定义但无调用者 |
| DNS 规则引擎 | 不支持 | 无清除/重建 API |
| DNS 缓存（含负缓存） | 不支持 | 无清除 API |
| 连接池参数 | 不支持 | 启动时初始化 |
| 监听端口 | 不支持 | 启动时绑定 |

配置修改的唯一生效方式是**重启进程**。

---

## 综合排障决策树

当出现连接异常时，按以下决策树快速定位根因。

```
症状：新连接超时或拒绝
│
├── 所有域名都超时？
│   ├── 是 → 检查 fd 使用
│   │        │
│   │        ├── fd 已耗尽（接近 ulimit）
│   │        │   ├── 恶意连接攻击？→ Session 无 idle timeout (P0)
│   │        │   ├── 连接池膨胀？  → 连接池无总容量上限 (P1)
│   │        │   └── socket 泄漏？  → 监听器 socket 泄漏 (P1)
│   │        │
│   │        └── fd 充足
│   │            └── accept 延迟高？→ EMFILE 退避未恢复 (P1)
│   │
│   └── 否（仅特定域名）→ 检查 DNS 缓存
│              │
│              ├── 日志有 "negative cache hit" → DNS 负缓存 TTL (P1)
│              │   └── 等待 TTL 过期 或 重启进程
│              │
│              └── 无负缓存 → 检查上游 DNS
│                  └── dig +trace example.com
│
├── 症状：进程无法停止
│   └── systemctl stop 超时 → 无优雅关闭机制 (P0)
│       └── 使用 SIGKILL 强制终止
│
├── 症状：连接建立但数据异常截断
│   └── 特定协议？→ 隧道半关闭语义丢失 (P2)
│
├── 症状：配置修改不生效
│   └── 配置热更新未实现 (P2)
│       └── 重启进程
│
└── 症状：无法获取运行指标
    └── 统计指标不对外暴露 (P2)
        └── 使用日志和系统工具替代
```

### 快速排障命令集

```bash
#!/bin/bash
# prism-quick-diag.sh — Prism 快速诊断脚本

PID=$(pgrep prism)
echo "=== Prism 进程诊断 (PID: $PID) ==="

echo -e "\n[1] fd 使用情况"
echo "进程 fd 数: $(lsof -p $PID 2>/dev/null | wc -l)"
echo "系统 fd: $(cat /proc/sys/fs/file-nr)"
echo "进程 fd 上限: $(cat /proc/$PID/limits 2>/dev/null | grep 'open files')"

echo -e "\n[2] TCP 连接状态"
ss -p | grep "pid=$PID" | awk '{print $1}' | sort | uniq -c | sort -rn

echo -e "\n[3] 内存使用"
cat /proc/$PID/status | grep -E 'VmRSS|VmSize|VmPeak'

echo -e "\n[4] 最近日志异常"
tail -1000 /var/log/prism.log | grep -c "EMFILE" && echo "次 EMFILE" || echo "无 EMFILE"
tail -1000 /var/log/prism.log | grep -c "negative cache hit" && echo "次负缓存命中" || echo "无负缓存命中"
tail -1000 /var/log/prism.log | grep -c "new connection" && echo "次新连接创建" || echo "无新连接"

echo -e "\n[5] 孤立 Session 检测（只有 started 无后续）"
STARTED=$(grep -c "Session.*started" /var/log/prism.log)
COMPLETED=$(grep -c "Session.*completed\|Session.*closed" /var/log/prism.log)
echo "已启动: $STARTED, 已完成: $COMPLETED, 差值: $((STARTED - COMPLETED))"

echo -e "\n=== 诊断完成 ==="
```

### 日志模式速查

| 日志模式 | 含义 | 对应风险 |
|----------|------|----------|
| 大量 `[Session] started` 无后续 | 空闲连接堆积 | Session 无 idle timeout (P0) |
| `EMFILE, delaying accept` | fd 耗尽 | 多种 fd 相关风险 |
| `negative cache hit` 连续出现 | DNS 负缓存生效 | DNS 负缓存 TTL (P1) |
| `new connection to {addr}` 高频 | 连接池膨胀 | 连接池无总容量上限 (P1) |
| accept 间隔 > 1s | 退避延迟未恢复 | EMFILE 指数退避 (P1) |
| 无 `shutting down` 日志 | 强制终止 | 无优雅关闭 (P0) |

---

## 风险依赖关系

各风险之间并非独立，它们通过 fd、协程调度、DNS 缓存等共享资源相互关联。

```
Session 无 idle timeout (P0)
    │
    ├── fd 累积 ──→ EMFILE ──→ EMFILE 指数退避不恢复 (P1)
    │                              │
    │                              └── 新连接接受速率低下
    │                                  → 用户感知为"服务挂了"
    │
    ├── fd 累积 ──→ 连接池无法创建新连接 ──→ 连接池相关风险 (P1)
    │
    └── fd 累积 ──→ DNS resolver 无法创建 UDP socket
                      → DNS 查询失败
                      → 负缓存写入 (P1)
                      → 更多域名受影响

无优雅关闭 (P0)
    │
    └── 强制终止 ──→ socket 泄漏 (P1)
                  ──→ 数据丢失
                  ──→ 重启后负缓存被清除（唯一的"正"面效果）
```

### 级联故障时间线

以下是一个典型的级联故障场景，展示了多个风险如何依次触发：

```
T+0h      正常运行
           fd 使用: 200/65535

T+2h      恶意客户端开始建立空闲连接
           [Session 无 idle timeout] → 连接永不释放

T+4h      fd 使用: 2000/65535
           连接池仍在正常工作

T+6h      fd 使用: 10000/65535
           DNS 偶尔创建 socket 失败
           [DNS 负缓存] → 部分域名被缓存为失败

T+7h      fd 使用: 30000/65535
           连接池创建新连接开始频繁失败
           [连接池无上限] → 无法阻止更多连接创建尝试

T+7.5h    fd 使用: 65535/65535 → EMFILE
           [EMFILE 指数退避] → accept 延迟增长到 5120ms
           新连接几乎无法建立

T+7.5h+   即使 fd 释放（session 自然结束），accept 仍被延迟
           [EMFILE 退避不恢复] → 服务降级持续

T+8h      运维尝试重启
           [无优雅关闭] → systemctl stop 超时
           → SIGKILL 强制终止
           [socket 泄漏] → 少量 fd 泄漏
           重启后恢复
```

---

## 修复优先级建议

| 优先级 | 风险 | 建议方案 | 预估工作量 |
|--------|------|----------|-----------|
| 1 | Session 无 idle timeout | 添加 idle timeout 定时器，预读阶段 30s，数据阶段 300s | 中 |
| 2 | EMFILE 指数退避不恢复 | accept 成功后重置 delay 为 min_delay | 小 |
| 3 | DNS 负缓存 TTL | 减小默认 negative_ttl 至 30s，增加清除 API | 小 |
| 4 | 无优雅关闭 | 实现信号处理 + io_context.stop() + drain | 大 |
| 5 | 连接池无总容量上限 | 添加全局 fd 计数器，超过阈值时拒绝新连接 | 中 |
| 6 | 统计指标暴露 | 添加 /metrics HTTP endpoint | 中 |
| 7 | 配置热更新 | 接入 swap_config() + DNS 规则重建 | 大 |
| 8 | 监听器 socket 泄漏 | 取消时显式关闭已 accept 的 socket | 小 |
| 9 | 隧道半关闭 | 修改为 drain 模式（可选） | 中 |
