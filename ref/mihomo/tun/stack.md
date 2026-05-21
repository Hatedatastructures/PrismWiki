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

## 三种 Stack 原理详解

### gVisor Stack：用户态网络栈

gVisor 是 Google 开发的容器沙箱项目，mihomo 使用了其中的用户态网络栈实现 (`gvisor.dev/gvisor/pkg/tcpip`)。

**架构：**
```
┌─────────────────────────────────────────────┐
│              应用程序                         │
│                  │                            │
│                  ▼                            │
│          系统网络协议栈                        │
│                  │                            │
│                  ▼                            │
│          TUN 虚拟网卡                          │
│                  │                            │
│                  ▼                            │
│  ┌───────────────────────────────────┐       │
│  │    mihomo (用户态进程)             │       │
│  │                                   │       │
│  │  ┌─────────────────────────────┐  │       │
│  │  │   gVisor tcpip Stack        │  │       │
│  │  │  ┌─────────────────────┐    │  │       │
│  │  │  │  Transport Layer    │    │  │       │
│  │  │  │  (TCP/UDP/ICMP)     │    │  │       │
│  │  │  ├─────────────────────┤    │  │       │
│  │  │  │  Network Layer      │    │  │       │
│  │  │  │  (IP routing, NDP)  │    │  │       │
│  │  │  ├─────────────────────┤    │  │       │
│  │  │  │  Link Layer         │    │  │       │
│  │  │  │  (TUN endpoint)     │    │  │       │
│  │  │  └─────────────────────┘    │  │       │
│  │  └─────────────────────────────┘  │       │
│  │                                   │       │
│  │  ┌─────────────────────────────┐  │       │
│  │  │   mihomo 代理逻辑             │  │       │
│  │  │   (规则匹配、代理选择)        │  │       │
│  │  └─────────────────────────────┘  │       │
│  └───────────────────────────────────┘       │
└─────────────────────────────────────────────┘
```

**数据包流转：**

1. 应用发送数据包到系统网络栈
2. 系统路由表将数据包路由到 TUN 接口
3. TUN 接口将数据包送入 mihomo 进程
4. gVisor 网络栈在用户态解析 IP 头、TCP/UDP 头
5. gVisor 构造 TCP 状态机（SYN/ACK/FIN 等）
6. mihomo 获取连接信息，执行代理逻辑
7. 代理响应数据通过 gVisor 写回 TUN 设备
8. 内核将数据传递给原始应用

**gVisor 网络栈核心组件：**

| 组件 | 职责 |
|------|------|
| `stack.Stack` | 网络栈主结构，管理所有协议层 |
| `tcpip.Endpoint` | 协议端点（TCP/UDP/RAW） |
| `tcpip NIC` | 网络接口卡抽象 |
| `tcpip.RouteTable` | 用户态路由表 |
| `TCP Forwarder` | TCP 连接转发器，处理入站连接 |
| `UDP Forwarder` | UDP 连接转发器 |

**TCP 状态机：**
gVisor 在用户态实现了完整的 TCP 状态机，包括：
- `SYN_SENT → SYN_RECEIVED → ESTABLISHED`（三次握手）
- `FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED`（四次挥手）
- 拥塞控制（Reno、CUBIC 等）
- 重传机制（RTO 计算、快速重传）

### System Stack：内核态网络栈

System Stack 利用操作系统的内核网络协议栈来处理 TCP/UDP 连接。

**架构（Linux）：**
```
┌─────────────────────────────────────────────┐
│              应用程序                         │
│                  │                            │
│                  ▼                            │
│          内核网络协议栈                        │
│          ┌─────────────────┐                  │
│          │ TCP/UDP 协议     │                  │
│          ├─────────────────┤                  │
│          │ IP 路由          │                  │
│          ├─────────────────┤                  │
│          │ TUN 驱动         │ ← mihomo 只读写  │
│          └─────────────────┘                  │
│                  │                            │
│                  ▼                            │
│  ┌───────────────────────────────────┐       │
│  │    mihomo (用户态进程)             │       │
│  │  仅做 IP 包级别的拦截和转发         │       │
│  └───────────────────────────────────┘       │
└─────────────────────────────────────────────┘
```

**工作原理：**

1. 使用 Linux 内核的 NAT (netfilter) 功能
2. 通过 `iptables`/`nftables` 规则将流量重定向
3. 内核处理完整的 TCP/UDP 协议逻辑
4. mihomo 只需要在 IP 包层面做拦截和转发

**优势：**
- 内核态 TCP 状态机，经过充分测试和优化
- 零拷贝路径（`splice`、`sendfile` 等）
- 硬件卸载支持（TSO、GRO、GSO）
- 无用户态-内核态上下文切换开销

**实现方式（Linux）：**
- 使用 `tun` 设备 + `iptables` NAT 规则
- `TPROXY` 透明代理重定向
- 需要 `CAP_NET_ADMIN` 权限

### Mixed Stack：混合模式

Mixed Stack 是 mihomo 的独创设计，结合两种栈的优势：

```
                    入站流量
                       │
                       ▼
              ┌────────────────┐
              │  协议类型判断   │
              └───────┬────────┘
                      │
            ┌─────────┴─────────┐
            │                   │
            ▼                   ▼
     TCP 流量 (IP 协议 6)   UDP 流量 (IP 协议 17)
            │                   │
            ▼                   ▼
     ┌─────────────┐     ┌─────────────┐
     │System Stack │     │ gVisor Stack│
     │ (内核态 TCP) │     │ (用户态 UDP)│
     └─────────────┘     └─────────────┘
```

**设计理由：**
- **TCP**: 内核态 TCP 状态机复杂且优化充分，system stack 性能更好
- **UDP**: 用户态 UDP 处理避免了内核态 socket 管理的复杂性，gVisor 实现更灵活

## 性能对比与分析

### 理论性能

| 指标 | System | Mixed | gVisor |
|------|--------|-------|--------|
| TCP 吞吐量 | 最高 | 高 | 中高 |
| UDP 延迟 | 中 | 低 | 最低 |
| CPU 占用 | 最低 | 低 | 中高 |
| 内存占用 | 最低 | 中 | 高 |
| 连接数上限 | 最高 | 高 | 中高 |

### 实际测试参考值

以下是在典型桌面环境下的参考值（千兆网络，i7 处理器）：

| Stack | 下载速度 | CPU 占用 | 延迟增加 |
|-------|----------|----------|----------|
| System | ~940 Mbps | ~2-3% | ~0.5 ms |
| Mixed | ~920 Mbps | ~3-4% | ~0.8 ms |
| gVisor | ~850 Mbps | ~5-8% | ~1.5 ms |

**注：** 实际值因硬件、系统版本、流量类型而异。

### 何时选择哪个 Stack

| 场景 | 推荐 | 理由 |
|------|------|------|
| 日常桌面使用 | gVisor | 兼容性好，无需特殊配置 |
| Linux 服务器 | System | 性能最优，内核支持好 |
| 游戏/低延迟 | System/Mixed | TCP 延迟敏感 |
| Windows/macOS | gVisor | 无内核依赖，稳定 |
| 高并发连接 | System | 内核连接管理更高效 |
| 特定 UDP 问题 | Mixed | TCP 性能 + UDP 灵活 |

## gVisor 网络栈与原始 gVisor 容器的关系

mihomo 使用的 gVisor 网络栈是 `gvisor.dev/gvisor/pkg/tcpip` 包，这是 gVisor 项目的核心网络协议栈库。

**关键区别：**

| 方面 | gVisor 容器 | mihomo gVisor Stack |
|------|-------------|---------------------|
| 用途 | 容器沙箱网络 | 代理透明代理 |
| 完整度 | 完整网络栈 | 仅使用 tcpip 包 |
| 端点管理 | 管理所有 NIC | 仅管理 TUN 端点 |
| 路由 | 完整路由表 | 简单 TUN 路由 |