---
title: "TUN stack 配置"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/tun
source: "mihomo-Memo/transport/tun/"
tags: [mihomo, tun, stack, gvisor, system, network]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/tun/overview]]"
  - "[[ref/mihomo/tun/enable]]"
---

# TUN stack 配置

**类别**: Mihomo TUN 配置 | **参数**: stack

## 配置格式

```yaml
tun:
  enable: true
  stack: gvisor     # 用户态网络栈（推荐）
  # stack: system   # 内核态网络栈
  # stack: mixed    # 混合模式
```

## Stack 类型对比

| Stack | 实现位置 | 性能 | 兼容性 | 适用平台 |
|-------|----------|------|--------|----------|
| gvisor | 用户态 | 中等 | 最好 | 所有平台 |
| system | 内核态 | 最高 | 受限 | Linux 优先 |
| mixed | 混合 | 中高 | 中等 | 特定场景 |

### gVisor Stack

```yaml
tun:
  stack: gvisor
```

**特点**:
- Google 开发的用户态网络栈
- 基于 Go 语言实现
- 不依赖系统内核特性
- 安全隔离，内核漏洞不影响

**优势**:
- 跨平台兼容性最好
- 稳定可靠
- 无需特殊权限
- Windows/macOS 默认选择

**劣势**:
- 性能略低于 system
- 用户态处理有额外开销

### System Stack

```yaml
tun:
  stack: system
```

**特点**:
- 使用系统内核的网络栈
- 内核态处理效率最高
- 需要系统权限和内核支持
- Linux 上需要 NET_ADMIN 能力

**优势**:
- 性能最高
- 内核态处理效率
- 系统原生实现

**劣势**:
- 需要更多权限
- 平台兼容性问题
- 受内核版本限制

**适用场景**:
- Linux 服务器
- 高流量环境
- 性能优先场景

### Mixed Stack

```yaml
tun:
  stack: mixed
```

**特点**:
- TCP 使用 system stack
- UDP 使用 gVisor stack
- 兼顾 TCP 性能和 UDP 兼容性

**适用场景**:
- UDP 兼容性问题
- TCP 性能优化需求
- 特定网络环境

## 选择建议

| 平台 | 推荐 Stack | 原因 |
|------|------------|------|
| Windows | gvisor | 兼容性最好，无内核依赖 |
| macOS | gvisor | 避免权限和 SIP 问题 |
| Linux | system | 性能最优，内核支持好 |
| Linux (兼容性) | gvisor | 内核版本限制场景 |
| UDP 问题场景 | mixed | TCP 性能 + UDP 兼容 |

## 性能对比

```
┌─────────────────────────────────────┐
│          Stack 性能对比              │
├─────────────────────────────────────┤
│                                      │
│  System    ████████████████ 100%    │
│  Mixed     ██████████████   85%     │
│  gVisor    ████████████     70%     │
│                                      │
└─────────────────────────────────────┘
```

注：实际性能差异取决于:
- 系统内核版本
- CPU 架构
- 流量类型 (TCP/UDP)
- 网络环境

## 配置示例

### 日常使用配置

```yaml
tun:
  enable: true
  stack: gvisor      # 推荐，兼容性好
  auto-route: true
  auto-detect-interface: true
```

### Linux 高性能配置

```yaml
tun:
  enable: true
  stack: system      # Linux 推荐
  auto-route: true
```

### UDP 优化配置

```yaml
tun:
  enable: true
  stack: mixed       # TCP system + UDP gvisor
```

## 源码位置

- gVisor 实现: `transport/tun/gvisor/`
- System 实现: `transport/tun/system/`
- Stack 选择: `transport/tun/tun.go`

## 相关链接

- [[overview]] — TUN 模式总览
- [[ref/mihomo/tun/enable|enable]] — 启用配置
- [[ref/network/tcp]] — TCP 协议
- [[ref/network/udp]] — UDP 协议