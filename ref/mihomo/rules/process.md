---
title: "mihomo 进程规则"
category: "mihomo"
type: ref
layer: ref
module: "rules/common"
source: "mihomo-Meta"
tags: [mihomo, rules, process, 进程规则]
created: 2026-05-17
updated: 2026-05-17
---

# mihomo 进程规则

进程规则根据发起请求的应用进程进行匹配（仅 TUN 或系统代理模式有效）。

## 源码位置

| 文件 | 描述 |
|------|------|
| `rules/common/process.go` | 进程规则实现 |

## Process 结构

```go
type Process struct {
    Base
    pattern  string
    adapter  string
    ruleType C.RuleType
    regexp   *regexp2.Regexp
}

func (ps *Process) Match(metadata *C.Metadata, helper C.RuleMatchHelper) (bool, string) {
    if helper.FindProcess != nil {
        helper.FindProcess()
    }
    var target string
    switch ps.ruleType {
    case C.ProcessName, C.ProcessNameRegex, C.ProcessNameWildcard:
        target = metadata.Process
    default:
        target = metadata.ProcessPath
    }
    
    switch ps.ruleType {
    case C.ProcessNameRegex, C.ProcessPathRegex:
        match, _ := ps.regexp.MatchString(target)
        return match, ps.adapter
    case C.ProcessNameWildcard, C.ProcessPathWildcard:
        return wildcard.Match(strings.ToLower(ps.pattern), strings.ToLower(target)), ps.adapter
    default:
        return strings.EqualFold(target, ps.pattern), ps.adapter
    }
}
```

## YAML 配置

### PROCESS-NAME（进程名匹配）

```yaml
rules:
  - PROCESS-NAME,chrome.exe,Proxy
  - PROCESS-NAME,firefox.exe,Proxy
  - PROCESS-NAME,steam.exe,DIRECT
  - PROCESS-NAME,spotify.exe,DIRECT
```

### PROCESS-PATH（进程路径匹配）

```yaml
rules:
  - PROCESS-PATH,C:\Program Files\Google\Chrome\Application\chrome.exe,Proxy
  - PROCESS-PATH,/usr/bin/curl,Proxy
```

### PROCESS-NAME-REGEX（进程名正则）

```yaml
rules:
  - PROCESS-NAME-REGEX,chrome.*,Proxy
  - PROCESS-NAME-REGEX,^steam,Direct
```

### PROCESS-PATH-REGEX（进程路径正则）

```yaml
rules:
  - PROCESS-PATH-REGEX,.*Chrome.*,Proxy
```

### PROCESS-NAME-WILDCARD（进程名通配符）

```yaml
rules:
  - PROCESS-NAME-WILDCARD,*chrome*,Proxy
```

## 配置字段说明

| 规则类型 | 格式 | 说明 |
|----------|------|------|
| `PROCESS-NAME` | `PROCESS-NAME,进程名,策略` | 进程名精确匹配 |
| `PROCESS-PATH` | `PROCESS-PATH,路径,策略` | 进程路径精确匹配 |
| `PROCESS-NAME-REGEX` | `PROCESS-NAME-REGEX,正则,策略` | 进程名正则匹配 |
| `PROCESS-PATH-REGEX` | `PROCESS-PATH-REGEX,正则,策略` | 进程路径正则匹配 |
| `PROCESS-NAME-WILDCARD` | `PROCESS-NAME-WILDCARD,通配符,策略` | 进程名通配符匹配 |
| `PROCESS-PATH-WILDCARD` | `PROCESS-PATH-WILDCARD,通配符,策略` | 进程路径通配符匹配 |

## 使用场景

### 按应用分流

```yaml
rules:
  # 浏览器走代理
  - PROCESS-NAME,chrome.exe,Proxy
  - PROCESS-NAME,firefox.exe,Proxy
  - PROCESS-NAME,edge.exe,Proxy
  
  # 游戏直连
  - PROCESS-NAME,steam.exe,DIRECT
  - PROCESS-NAME,LeagueofLegends.exe,DIRECT
  - PROCESS-NAME,dota2.exe,DIRECT
  
  # 开发工具
  - PROCESS-NAME,idea64.exe,开发服务
  - PROCESS-NAME,vscode.exe,开发服务
  - PROCESS-NAME,git.exe,开发服务
  
  # 广告拦截
  - PROCESS-NAME,adb.exe,REJECT
```

### 注意事项

1. **仅 TUN/系统代理模式有效**：进程规则需要在 TUN 或系统代理模式下才能生效
2. **平台差异**：
   - Windows：进程名通常带 `.exe` 后缀
   - Linux/macOS：进程名不带后缀
3. **权限要求**：需要足够的权限才能获取进程信息

## 相关文档

- [[overview]] - 规则系统概述
- [[port]] - 端口规则
- [[ref/mihomo/tun|TUN 模式]]