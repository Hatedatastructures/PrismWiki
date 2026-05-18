---
title: "Rule Script"
category: "mihomo"
type: ref
layer: ref
module: ref/mihomo
source: "mihomo-Meta/script"
tags: [mihomo, script, rule-script, 规则脚本, lua, python]
created: 2026-05-17
updated: 2026-05-17
related: [overview, shortcuts]
---

# Rule Script

**类别**: Mihomo 脚本功能

## 概述

Rule Script（规则脚本）允许通过脚本动态处理连接路由。脚本接收连接信息，返回路由决策，实现自定义规则逻辑。

## 基础配置

### Python 脚本

```yaml
script:
  code: |
    def main(ctx, metadata):
        host = metadata.get("host", "")
        if host.endswith(".cn"):
            return "DIRECT"
        return "PROXY"
```

### Lua 脚本

```yaml
script:
  code: |
    function main(ctx, metadata)
        host = metadata["host"]
        if host:match("%.cn$") then
            return "DIRECT"
        end
        return "PROXY"
    end
```

### 文件脚本

```yaml
script:
  path: ./script/rules.py
```

## 配置参数

### code

```yaml
code: |
  def main(ctx, metadata):
      # 脚本逻辑
      return "PROXY"
```

内嵌脚本代码。

### path

```yaml
path: ./script/rules.py
```

脚本文件路径。

### 主函数

脚本必须定义 `main` 函数：

```python
def main(ctx, metadata):
    # ctx: 上下文对象
    # metadata: 连接信息
    # return: 路由决策
    return "PROXY"
```

## Metadata 参数

metadata 包含连接信息：

| 字段 | 类型 | 说明 |
|------|------|------|
| host | string | 目标主机名 |
| process | string | 进程名 |
| process_path | string | 进程路径 |
| src_ip | string | 来源 IP |
| src_port | int | 来源端口 |
| dst_ip | string | 目标 IP |
| dst_port | int | 目标端口 |
| network | string | 网络类型 (tcp/udp) |
| type | string | 入站类型 |

## 返回值

脚本返回路由决策：

| 返回值 | 说明 |
|--------|------|
| DIRECT | 直连 |
| REJECT | 拒绝 |
| PROXY | 使用默认代理 |
| 代理组名称 | 使用指定代理组 |

## 在规则中使用

```yaml
script:
  code: |
    def main(ctx, metadata):
        host = metadata.get("host", "")
        if host.endswith(".cn"):
            return "DIRECT"
        return "PROXY"

rules:
  - SCRIPT,my-script,PROXY
```

SCRIPT 规则格式：`SCRIPT,脚本名称,默认策略`

## 脚本示例

### 域名分流

```yaml
script:
  code: |
    def main(ctx, metadata):
        host = metadata.get("host", "")
        
        # 国内域名直连
        if host.endswith(".cn") or host.endswith(".com.cn"):
            return "DIRECT"
        
        # 广告域名拒绝
        if "ads" in host or "ad" in host:
            return "REJECT"
        
        # 其他走代理
        return "PROXY"

rules:
  - SCRIPT,domain-rule,PROXY
```

### 进程分流

```yaml
script:
  code: |
    def main(ctx, metadata):
        process = metadata.get("process", "")
        
        # 浏览器走代理
        if process in ["chrome", "firefox", "edge"]:
            return "PROXY"
        
        # 系统进程直连
        if process in ["system", "svchost"]:
            return "DIRECT"
        
        # 其他默认
        return "PROXY"

rules:
  - SCRIPT,process-rule,PROXY
```

### 源 IP 分流

```yaml
script:
  code: |
    def main(ctx, metadata):
        src_ip = metadata.get("src_ip", "")
        
        # 内网直连
        if src_ip.startswith("192.168.") or src_ip.startswith("10."):
            return "DIRECT"
        
        # 外网走代理
        return "PROXY"

rules:
  - SCRIPT,ip-rule,PROXY
```

### 时间分流

```yaml
script:
  code: |
    import datetime
    
    def main(ctx, metadata):
        now = datetime.datetime.now()
        hour = now.hour
        
        # 工作时间走代理
        if 9 <= hour < 18:
            return "PROXY"
        
        # 其他时间直连
        return "DIRECT"

rules:
  - SCRIPT,time-rule,PROXY
```

### 代理组选择

```yaml
script:
  code: |
    def main(ctx, metadata):
        host = metadata.get("host", "")
        
        # 流媒体使用特定代理组
        if host in ["youtube.com", "netflix.com"]:
            return "Streaming"
        
        # 社交媒体使用特定代理组
        if host in ["twitter.com", "facebook.com"]:
            return "Social"
        
        # 其他默认
        return "Auto"

proxy-groups:
  - name: Streaming
    type: select
    proxies: [节点A, 节点B]
  - name: Social
    type: select
    proxies: [节点C, 覃点D]
  - name: Auto
    type: url-test
    proxies: [节点A, 覃点B, 覃点C, 覃点D]

rules:
  - SCRIPT,group-rule,Auto
```

### Lua 脚本示例

```yaml
script:
  code: |
    function main(ctx, metadata)
        host = metadata["host"] or ""
        process = metadata["process"] or ""
        
        -- 国内域名直连
        if host:match("%.cn$") then
            return "DIRECT"
        end
        
        -- Chrome 走代理
        if process == "chrome" then
            return "PROXY"
        end
        
        -- 默认
        return "PROXY"
    end

rules:
  - SCRIPT,lua-rule,PROXY
```

## 上下文对象

ctx 提供额外功能：

| 方法 | 说明 |
|------|------|
| ctx.resolve_ip(host) | 解析 IP |
| ctx.geoip(ip) | GEOIP 查询 |
| ctx.log(message) | 日志输出 |

```yaml
script:
  code: |
    def main(ctx, metadata):
        host = metadata.get("host", "")
        ip = ctx.resolve_ip(host)
        
        if ip:
            country = ctx.geoip(ip)
            if country == "CN":
                return "DIRECT"
        
        return "PROXY"
```

## 性能考量

脚本执行开销：

| 因素 | 影响 |
|------|------|
| 脚本复杂度 | 执行时间 |
| 外部调用 | IO 延迟 |
| 每连接执行 | 累计开销 |

建议：
- 简化脚本逻辑
- 减少外部调用
- 使用内置规则优先

## 排错指南

### 脚本不生效

检查项：

1. 主函数是否正确定义
2. 返回值是否有效
3. SCRIPT 规则是否配置

### 返回值错误

确保返回值：

- DIRECT / REJECT / PROXY
- 或已定义的代理组名称

### 性能问题

优化建议：

- 简化逻辑
- 避免复杂计算
- 使用缓存

## 相关链接

- [[ref/mihomo/script/overview|Script 概览]] — 脚本功能总览
- [[ref/mihomo/script/shortcuts|Shortcuts]] — 节点快捷配置
- [[ref/mihomo/rules/overview|rules]] — 规则配置参考