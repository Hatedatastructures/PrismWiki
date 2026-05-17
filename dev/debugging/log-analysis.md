---
title: 日志分析方法指南
source:
  - I:/code/Prism/include/prism/trace/
  - I:/code/Prism/src/prism/trace/
module: debugging
type: reference
tags: [logging, analysis, trace, debug, grep, filter, pattern]
layer: dev
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/debugging/overview]]"
  - "[[trace]]"
  - "[[dev/debugging/common-issues]]"
---

# 日志分析方法指南

日志是排障的核心依据。本文档介绍日志结构、分析方法、常用命令和过滤技巧。

## 概述

### 日志结构

Prism 日志采用标准格式：

```
[时间戳] [标识] [级别] 消息内容
```

示例：
```
[2026-05-17 10:30:45.123] [prism] [info] Session started: uuid=xxx
```

### 日志级别层次

| 级别 | 数值 | 含义 | 使用场景 |
|------|------|------|----------|
| trace | 0 | 最详细 | 详细跟踪 |
| debug | 1 | 调试信息 | 问题排查 |
| info | 2 | 常规信息 | 运行状态 |
| warn | 3 | 警告 | 异常情况 |
| error | 4 | 错误 | 错误事件 |
| critical | 5 | 严重 | 致命错误 |

### 日志内容分类

| 分类 | 内容 | 级别 |
|------|------|------|
| 生命周期 | 启动、停止、会话 | info |
| 网络事件 | 连接、断开、传输 | info/debug |
| 协议处理 | 识别、解析、认证 | debug |
| 错误信息 | 失败、异常 | error/warn |
| 性能统计 | 延迟、吞吐 | debug |

---

## 详解

### 基本日志分析

#### 查看日志文件

```bash
# 查看最新日志
tail -f prism.log

# 查看历史日志
cat prism.log

# 分页查看
less prism.log
```

#### 日志级别过滤

```bash
# 只看错误日志
grep "\[error\]" prism.log

# 只看警告和错误
grep -E "\[error\]|\[warn\]" prism.log

# 排除调试日志
grep -v "\[debug\]" prism.log

# 只看 info 级别
grep "\[info\]" prism.log
```

#### 时间范围过滤

```bash
# 查看特定时间段
grep "2026-05-17 10:30" prism.log

# 查看特定日期
grep "2026-05-17" prism.log

# 查看最近一小时（假设当前为 11:00）
grep "2026-05-17 10:[0-5]" prism.log
```

### 关键事件分析

#### 启动和停止

```bash
# 查看启动日志
grep -E "started|Starting|initialized" prism.log

# 查看停止日志
grep -E "stopped|Stopping|shutdown" prism.log
```

#### 连接事件

```bash
# 新连接
grep "New connection" prism.log

# 连接断开
grep -E "closed|disconnect|timeout" prism.log

# 连接失败
grep "Connection failed" prism.log
```

#### 会话事件

```bash
# 会话创建
grep "Session created" prism.log

# 会话销毁
grep "Session closed" prism.log

# 会话计数
grep "Session count" prism.log
```

#### 认证事件

```bash
# 认证成功
grep "Authentication success" prism.log

# 认证失败
grep "Authentication failed" prism.log

# 用户查找
grep -E "User found|User not found" prism.log
```

### 错误分析

#### 错误统计

```bash
# 统计各类型错误
grep "\[error\]" prism.log | awk -F'code=' '{print $2}' | sort | uniq -c

# 统计错误数量
grep "\[error\]" prism.log | wc -l

# 错误类型分布
grep "\[error\]" prism.log | grep -oP 'code=\w+' | sort | uniq -c
```

#### 错误详情查看

```bash
# 查看完整错误信息
grep "\[error\]" prism.log | head -20

# 查看特定错误码
grep "code=timeout" prism.log

# 查看错误上下文（前后 3 行）
grep -B3 -A3 "\[error\]" prism.log
```

#### 错误趋势分析

```bash
# 按小时统计错误
grep "\[error\]" prism.log | cut -d']' -f1 | cut -d' ' -f2 | cut -d':' -f1 | uniq -c

# 按天统计错误
grep "\[error\]" prism.log | cut -d' ' -f1 | uniq -c
```

### 性能分析

#### 延迟分析

```bash
# 查看延迟日志
grep "latency" prism.log

# 高延迟警告
grep "High latency" prism.log

# 延迟统计
grep "latency=" prism.log | grep -oP 'latency=\d+' | cut -d= -f2 | awk '{sum+=$1; count++} END {print "avg:", sum/count}'
```

#### 吞吐分析

```bash
# 查看吞吐日志
grep "Throughput" prism.log

# 查看请求统计
grep "requests=" prism.log

# 查看传输字节
grep "bytes=" prism.log
```

#### 资源使用

```bash
# CPU 使用
grep "CPU usage" prism.log

# 内存使用
grep "Memory usage" prism.log

# 连接数
grep -E "connections=|Session count" prism.log
```

---

## 日志分析脚本

### 错误分析脚本

```bash
#!/bin/bash
# analyze_errors.sh

LOG_FILE=$1

echo "=== Error Analysis ==="
echo ""

echo "Total errors:"
grep "\[error\]" $LOG_FILE | wc -l

echo ""
echo "Error types distribution:"
grep "\[error\]" $LOG_FILE | grep -oP 'code=\w+' | sort | uniq -c | sort -rn

echo ""
echo "Errors by hour:"
grep "\[error\]" $LOG_FILE | grep -oP '\d{2}:\d{2}' | cut -d: -f1 | sort | uniq -c

echo ""
echo "Top 10 error messages:"
grep "\[error\]" $LOG_FILE | head -10
```

### 会话分析脚本

```bash
#!/bin/bash
# analyze_sessions.sh

LOG_FILE=$1

echo "=== Session Analysis ==="
echo ""

echo "Sessions created:"
grep "Session created" $LOG_FILE | wc -l

echo "Sessions closed:"
grep "Session closed" $LOG_FILE | wc -l

echo ""
echo "Active sessions (created but not closed):"
CREATED=$(grep "Session created" $LOG_FILE | wc -l)
CLOSED=$(grep "Session closed" $LOG_FILE | wc -l)
echo "Active: $CREATED - $CLOSED = $(($CREATED - $CLOSED))"

echo ""
echo "Session duration (if logged):"
grep "Session closed" $LOG_FILE | grep -oP 'duration=\d+' | cut -d= -f2 | awk '{sum+=$1; count++} END {print "avg duration:", sum/count, "s"}'
```

### 连接分析脚本

```bash
#!/bin/bash
# analyze_connections.sh

LOG_FILE=$1

echo "=== Connection Analysis ==="
echo ""

echo "Total connections:"
grep "New connection" $LOG_FILE | wc -l

echo "Connection failures:"
grep "Connection failed" $LOG_FILE | wc -l

echo ""
echo "Failure reasons:"
grep "Connection failed" $LOG_FILE | grep -oP 'code=\w+' | sort | uniq -c | sort -rn

echo ""
echo "Connections by source IP:"
grep "New connection" $LOG_FILE | grep -oP 'from=[\d.]+' | cut -d= -f2 | sort | uniq -c | sort -rn | head -10
```

### 性能分析脚本

```bash
#!/bin/bash
# analyze_performance.sh

LOG_FILE=$1

echo "=== Performance Analysis ==="
echo ""

echo "High latency events:"
grep "High latency" $LOG_FILE | wc -l

echo ""
echo "Average latency (if logged):"
grep "latency=" $LOG_FILE | grep -oP 'latency=\d+' | cut -d= -f2 | awk '{sum+=$1; count++} END {print "avg:", sum/count, "ms"}'

echo ""
echo "CPU warnings:"
grep "CPU" $LOG_FILE | grep -E "warn|error" | wc -l

echo ""
echo "Memory warnings:"
grep "Memory" $LOG_FILE | grep -E "warn|error" | wc -l
```

---

## 日志过滤技巧

### 多条件过滤

```bash
# 同时过滤多个条件
grep -E "error|warn" prism.log | grep "timeout"

# 排除多个条件
grep -v -E "debug|trace" prism.log

# 组合过滤
grep "\[error\]" prism.log | grep "network" | grep "timeout"
```

### 正则表达式过滤

```bash
# IP 地址
grep -oP '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}' prism.log

# UUID
grep -oP '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' prism.log

# 端口号
grep -oP 'port=\d+' prism.log

# 错误码
grep -oP 'code=\w+' prism.log
```

### 字段提取

```bash
# 提取时间戳
grep -oP '\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}' prism.log

# 提取级别
grep -oP '\[\w+\]' prism.log | sort | uniq -c

# 提取消息类型
awk -F'] ' '{print $NF}' prism.log | cut -d':' -f1 | sort | uniq -c | sort -rn
```

---

## 实时日志监控

### 基本监控

```bash
# 实时查看
tail -f prism.log

# 实时过滤错误
tail -f prism.log | grep --line-buffered "\[error\]"

# 实时统计
tail -f prism.log | awk '{print $1, $2}' | uniq -c
```

### 高级监控脚本

```bash
#!/bin/bash
# monitor_log.sh

LOG_FILE=$1

echo "Real-time log monitoring..."

tail -f $LOG_FILE | while read line; do
    # 提取级别
    LEVEL=$(echo "$line" | grep -oP '\[\w+\]' | head -2 | tail -1)
    
    # 根据级别高亮
    case $LEVEL in
        "[error]")    echo "$(tput setaf 1)$line$(tput sgr0)" ;;
        "[warn]")     echo "$(tput setaf 3)$line$(tput sgr0)" ;;
        "[info]")     echo "$(tput setaf 2)$line$(tput sgr0)" ;;
        "[debug]")    echo "$line" ;;
        *)            echo "$line" ;;
    esac
done
```

---

## 最佳实践

### 日志分析建议

| 场景 | 建议方法 |
|------|----------|
| 快速定位错误 | grep error + head |
| 统计分析 | awk + sort + uniq |
| 趋势分析 | 按时间分组统计 |
| 实时监控 | tail -f + grep |

### 日志保留建议

```json
{
    "trace": {
        "max_size": 134217728,  // 128MB
        "max_files": 10,        // 10 个轮转文件
        "enable_file": true
    }
}
```

### 日志级别建议

| 环境 | 级别 | 原因 |
|------|------|------|
| 生产 | info | 平衡信息量 |
| 排障 | debug | 获取详情 |
| 开发 | trace | 最详细 |

---

## 常见问题

### Q1: 日志文件太大怎么办？

**A**: 
```bash
# 分割日志
split -l 100000 prism.log prism_log_

# 只看最近部分
tail -10000 prism.log > recent.log

# 使用压缩
gzip prism.log.old
```

### Q2: 如何快速找到特定时间段？

**A**: 
```bash
# 使用 awk 过滤时间范围
awk '/2026-05-17 10:30/,/2026-05-17 10:40/' prism.log

# 使用 grep
grep "2026-05-17 10:[3-4]" prism.log
```

### Q3: 如何分析日志趋势？

**A**: 
```bash
# 按小时统计事件数
grep "\[info\]" prism.log | grep -oP '\d{2}:\d{2}' | cut -d: -f1 | sort | uniq -c

# 按天统计
grep "\[info\]" prism.log | cut -d' ' -f1 | sort | uniq -c
```

### Q4: 如何导出日志数据？

**A**: 
```bash
# 导出错误日志
grep "\[error\]" prism.log > errors.log

# 导出 CSV 格式
awk -F'] ' '{print $1","$2","$3","$4}' prism.log > log.csv
```

---

## 相关链接

- [[dev/debugging/overview]] — 排障体系概述
- [[trace]] — Trace 日志模块
- [[dev/debugging/common-issues]] — 常见问题汇总
- [[dev/debugging/connection]] — 连接问题排查
- [[dev/debugging/protocol]] — 协议问题排查