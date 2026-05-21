---
title: "TUN enable 配置"
layer: ref
category: "mihomo"
type: ref
module: ref/mihomo/tun
source: "mihomo-Memo/transport/tun/tun.go"
tags: [mihomo, tun, enable, platform, driver]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[ref/mihomo/tun/overview]]"
  - "[[ref/mihomo/tun/stack]]"
  - "[[ref/mihomo/tun/auto-route]]"
---

# TUN enable 配置

**类别**: Mihomo TUN 配置 | **参数**: enable

## 配置格式

```yaml
tun:
  enable: true     # 启用 TUN 模式
  # enable: false  # 禁用 TUN 模式（默认）
```

## 参数说明

| 值 | 说明 |
|-----|------|
| true | 启用 TUN 虚拟网卡，开始透明代理 |
| false | 禁用 TUN 模式，不创建虚拟网卡 |

## 平台配置要求

### Windows

```yaml
tun:
  enable: true
  stack: gvisor
  auto-route: true
  auto-detect-interface: true
```

**依赖**:
- 需要 `wintun.dll` 文件（放在 mihomo 同目录）
- 需要管理员权限运行
- 会自动创建 "Clash" 网络适配器

### macOS

```yaml
tun:
  enable: true
  stack: gvisor
```

**依赖**:
- 使用系统原生 utun 驱动
- 可能需要 root 权限
- 注意 SIP (System Integrity Protection) 限制

### Linux

```yaml
tun:
  enable: true
  stack: system    # 推荐 system stack
```

**依赖**:
- `/dev/net/tun` 设备必须存在
- 需要 CAP_NET_ADMIN 能力
- systemd 配置示例:

```ini
[Service]
AmbientCapabilities=CAP_NET_ADMIN
```

### Android/iOS

移动端通过系统 API 实现:
- Android: VpnService API
- iOS: Network Extension framework

## 启用流程

```
enable: true
    │
    ▼
┌─────────────────────────────────────┐
│  检查平台权限                        │
│  Windows: 管理员                    │
│  Linux: CAP_NET_ADMIN               │
│  macOS: root                        │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  创建 TUN 设备                       │
│  Windows: WinTUN                    │
│  macOS: utun                        │
│  Linux: /dev/net/tun                │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  配置 Stack                          │
│  gVisor / System / Mixed            │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  配置路由 (auto-route)               │
│  添加路由规则                        │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  开始处理流量                        │
└─────────────────────────────────────┘
```

## 常见问题

### 启动失败

**Windows**:
```powershell
# 检查 wintun.dll
dir wintun.dll

# 检查管理员权限
# 需要以管理员身份运行
```

**Linux**:
```bash
# 检查 TUN 设备
ls -la /dev/net/tun

# 如果不存在，创建设备
mkdir -p /dev/net && mknod /dev/net/tun c 10 200

# 检查权限
chmod 666 /dev/net/tun
```

**macOS**:
```bash
# 检查 SIP 状态
csrutil status

# 可能需要特殊签名
```

## 相关链接

- [[overview]] — TUN 模式总览
- [[ref/mihomo/tun/stack|stack]] — 网络栈选择
- [[ref/mihomo/tun/auto-route|auto-route]] — 自动路由配置
- [[ref/mihomo/tun/overview|tun]] — 完整 TUN 说明

## TUN 模式原理

### 什么是 TUN

TUN（Tunnel）是一种操作系统内核提供的虚拟网络设备。它工作在 OSI 模型的**网络层（Layer 3）**，能够接收和发送原始 IP 数据包。

```
┌─────────────────────────────────────────────────────┐
│                    应用程序                          │
│         (浏览器、游戏、系统等)                        │
│                        │                            │
│                        ▼                            │
│              系统网络协议栈                           │
│                        │                            │
│                        ▼                            │
│              ┌─────────────────┐                    │
│              │   TUN 虚拟网卡    │ ← 关键点           │
│              └────────┬────────┘                    │
│                       │                              │
│                       ▼                              │
│         用户态程序 (mihomo) 读取 IP 包               │
└─────────────────────────────────────────────────────┘
```

TUN 与物理网卡的关键区别：

| 特性 | 物理网卡 | TUN 虚拟网卡 |
|------|----------|--------------|
| 数据来源 | 物理线路上的电信号/光信号 | 内核网络栈路由过来的 IP 包 |
| 数据格式 | 以太网帧（Layer 2） | 原始 IP 包（Layer 3） |
| 读写方式 | 驱动层接口 | 文件描述符（`/dev/net/tun` 等） |
| MAC 地址 | 有 | 无（Layer 3 设备） |

### TUN 透明代理工作原理

mihomo 的 TUN 模式通过以下步骤实现透明代理：

1. **创建虚拟网卡**: 在系统中创建一个 TUN 虚拟网络接口
2. **修改路由表**: 将系统默认路由指向 TUN 接口（通过拆分路由 `0.0.0.0/1 + 128.0.0.0/1`）
3. **劫持 DNS**: 拦截所有向端口 53 的 DNS 查询，交由 mihomo 的 DNS 模块处理
4. **读取 IP 包**: mihomo 从 TUN 设备读取所有 IP 数据包
5. **协议识别**: 分析数据包，识别目标地址和协议
6. **代理转发**: 根据规则决定直连或代理，通过代理服务器转发流量
7. **响应返回**: 将代理服务器返回的数据写回 TUN 设备，由内核传递给应用

## 内核实现：平台差异

### Linux: `/dev/net/tun`

Linux 的 TUN/TAP 驱动是内核模块，通过字符设备 `/dev/net/tun` 暴露给用户态。

```c
// Linux 内核 TUN 创建流程（简化）
int tun_fd = open("/dev/net/tun", O_RDWR);  // 打开字符设备

struct ifreq ifr = {0};
ifr.ifr_flags = IFF_TUN | IFF_NO_PI;  // TUN 模式，不包协议信息
strcpy(ifr.ifr_name, "tun0");

ioctl(tun_fd, TUNSETIFF, &ifr);  // 设置接口名称和模式
// 此后可以从 tun_fd 读取/写入原始 IP 包
```

**内核数据结构：**
```
sk_buff (内核网络包缓冲区)
    │
    ▼
tun_struct (TUN 接口结构)
    │
    ▼
socket->sk_receive_queue (接收队列)
    │
    ▼
用户态 read() ← 从这里取数据
```

**关键系统调用：**
- `open("/dev/net/tun")`: 创建设备文件描述符
- `ioctl(TUNSETIFF)`: 配置接口
- `ioctl(TUNSETPERSIST)`: 设置为持久模式
- `ioctl(TUNSETOWNER)`: 设置所有者
- `ioctl(TUNSETGROUP)`: 设置用户组

**权限要求：**
- `CAP_NET_ADMIN` 能力
- 或 `/dev/net/tun` 的读写权限

### Windows: WinTUN / Wintun.dll

Windows 没有原生的 TUN 驱动，mihomo 使用 WireGuard 项目的 **WinTUN** 实现。

**WinTUN 架构：**
```
用户态程序
    │
    │  Ring3/Ring0 共享内存
    ▼
┌─────────────────────────────────────┐
│  wintun.dll (用户态 DLL)             │
│  ├── 共享内存管理                     │
│  ├── 会话管理                        │
│  └── IOCTL 封装                     │
└────────────────┬────────────────────┘
                 │  DeviceIoControl
                 ▼
┌─────────────────────────────────────┐
│  wintun.sys (内核态驱动)             │
│  ├── NDIS 轻型驱动                    │
│  ├── 虚拟网络适配器                   │
│  └── 共享内存映射                     │
└─────────────────────────────────────┘
```

**WinTUN 核心机制：**

1. **共享内存环**: WinTUN 使用用户态和内核态共享的内存环形缓冲区来传递数据包，避免了传统的 `DeviceIoControl` 系统调用开销
2. **NDIS 轻型驱动**: 注册为 NDIS (Network Driver Interface Specification) 轻型驱动，系统将其视为一个真实的网络适配器
3. **会话 (Session)**: 通过 `WintunStartSession` 创建会话，获取读写句柄

```c
// WinTUN 创建流程（简化）
HMODULE wintun = LoadLibrary("wintun.dll");
WINTUN_CREATE_ADAPTER_FUNC *CreateAdapter = ...;

WINTUN_ADAPTER_HANDLE adapter = CreateAdapter(
    L"Clash",           // 适配器名称
    L"Wintun",          // 适配器类型
    NULL,               // 可选 GUID
    &adapter_guid
);

DWORD session = WintunStartSession(adapter, 0x400000, FALSE);
// 通过 WintunAllocateSendPacket / WintunReceivePacket 收发数据
```

**依赖文件：**
- `wintun.dll` 必须放在 mihomo 可执行文件同目录
- 需要管理员权限运行（注册内核驱动）
- 驱动签名验证：Windows 10/11 要求驱动有有效签名

**Windows 网络适配器管理：**
- 自动创建的适配器名称通常为 "Clash"
- 可以在网络连接中看到
- IP 地址通常设置为 `198.18.0.1/16`

### macOS: utun (Network Extension)

macOS 使用内核的 utun 框架，通过 `sys` 控制字 (control socket) 创建。

```c
// macOS utun 创建流程（简化）
int utun_fd = socket(AF_SYSTEM, SOCK_DGRAM, SYSPROTO_CONTROL);

struct ctl_info info;
strlcpy(info.ctl_name, "com.apple.net.utun_control", sizeof(info.ctl_name));
ioctl(utun_fd, CTLIOCGINFO, &info);

struct sockaddr_ctl addr;
addr.sc_len = sizeof(addr);
addr.sc_family = AF_SYSTEM;
addr.ss_sysaddr = AF_SYS_CONTROL;
addr.sc_id = info.ctl_id;
addr.sc_unit = 0;  // 自动分配 utun 编号

connect(utun_fd, (struct sockaddr*)&addr, sizeof(addr));
// 此后可以从 utun_fd 读取/写入原始 IP 包
```

**macOS 平台特殊性：**

| 特性 | 说明 |
|------|------|
| SIP 限制 | System Integrity Protection 可能限制某些操作 |
| 代码签名 | 可能需要特殊的代码签名授权 |
| utun 编号 | `sc_unit = 0` 自动分配，也可指定具体编号 |
| 权限 | 通常需要 root 或特定授权 |

**utun 数据格式：**
- macOS utun 在每个 IP 包前会附加一个 4 字节的协议类型头（`AF_INET` 或 `AF_INET6`）
- 读写时需要跳过这 4 个字节

## 平台对比总结

| 特性 | Linux | Windows (WinTUN) | macOS (utun) |
|------|-------|-------------------|--------------|
| 设备类型 | 字符设备 | NDIS 轻型驱动 | 控制字 Socket |
| 创建方式 | `open("/dev/net/tun")` | `LoadLibrary("wintun.dll")` | `socket(AF_SYSTEM)` |
| 数据格式 | 纯 IP 包 | 纯 IP 包 | 4 字节协议头 + IP 包 |
| 权限 | `CAP_NET_ADMIN` | 管理员 | root |
| 性能 | 最高（内核原生） | 高（共享内存） | 中（系统调用） |
| 依赖 | 内核模块 | wintun.dll | 系统框架 |
| 多实例 | 支持 | 支持 | 支持 |