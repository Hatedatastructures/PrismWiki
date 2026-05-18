---
title: 协议处理流程详解
created: 2026-05-17
layer: core
tags: [architecture, flow]
source: CLAUDE.md §协议处理流程
---

# 协议处理流程详解

从监听器接收连接到隧道转发的完整流程。

## 流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│  listener 接受连接                                                   │
│     ↓                                                               │
│  计算亲和性哈希 → 选择 worker                                         │
│     ↓                                                               │
│  balancer 分发 socket 到 worker                                      │
│     ↓                                                               │
│  worker 投递到 io_context → launch 创建会话                          │
│     ↓                                                               │
│  session 调用 recognition::recognize()                              │
│     ↓                                                               │
│  ┌─────────────────────────────────────────┐                        │
│  │  Recognition 三阶段流水线                 │                        │
│  │  ┌─────────────────────────────────────┐ │                        │
│  │  │ 1. Probe: 预读 24 字节               │ │                        │
│  │  │    检测 HTTP/SOCKS5/TLS/SS2022       │ │                        │
│  │  │    ↓                                 │ │                        │
│  │  │ 2. Identify (仅 TLS):                │ │                        │
│  │  │    read_clienthello()                │ │                        │
│  │  │    parse_clienthello()               │ │                        │
│  │  │    analyzer_registry::analyze()      │ │                        │
│  │  │    ↓                                 │ │                        │
│  │  │ 3. Execute:                          │ │                        │
│  │  │    scheme_executor::execute()        │ │                        │
│  │  │    → 返回 transport + detected       │ │                        │
│  │  └─────────────────────────────────────┘ │                        │
│  └─────────────────────────────────────────┘                        │
│     ↓                                                               │
│  dispatch 根据 detected 类型获取 handler                             │
│     ↓                                                               │
│  handler 调用 pipeline 处理                                          │
│     ↓                                                               │
│  pipeline 通过 router 建立上游连接                                    │
│     ↓                                                               │
│  tunnel 执行双向透明转发                                              │
└─────────────────────────────────────────────────────────────────────┘
```

## 各步骤详解

### Step 1: listener 接受连接

listener 监听配置的端点，接受 TCP 连接：

```cpp
socket = acceptor.accept();
hash = affinity_hash(socket.remote_endpoint());
```

计算客户端地址的哈希值，用于分发到固定 worker。

详见 [[core/agent/front/listener|监听器]]

### Step 2: balancer 分发连接

balancer 根据哈希选择 worker：

```cpp
worker = select(hash);
worker.post(socket);
```

分发 socket 到选中的 worker 的 io_context。

详见 [[core/agent/front/balancer|负载均衡器]]

### Step 3: launch 创建会话

worker 收到 socket 后调用 launch：

```cpp
launch(socket, config, tls_context, account_directory)
```

创建 session 对象，开始会话生命周期。

详见 [[core/agent/worker/launch|会话启动]]

### Step 4: Recognition 协议识别

session 调用 recognition::recognize()，三阶段流水线：

#### Probe（探测）

预读 24 字节，检测协议类型：

- HTTP: 以 "GET", "POST", "CONNECT" 开头
- SOCKS5: 首字节 0x05
- TLS: 首字节 0x16（Handshake）
- SS2022: 排除法 fallback

详见 [[core/recognition/probe/probe|协议探测]]

#### Identify（识别，仅 TLS）

读取 TLS ClientHello，特征分析：

```cpp
raw = read_clienthello(transport);
features = parse_clienthello(raw);
result = analyzer_registry::analyze(features);
```

返回候选方案列表和置信度。

详见 [[core/recognition/layered_pipeline|分层检测管道]]

#### Execute（执行）

根据分析结果执行伪装方案：

```cpp
execution = scheme_executor::execute(transport, candidate);
```

返回处理后的 transport 和检测到的协议类型。

详见 [[core/stealth/executor|方案执行器]]

### Step 5: Dispatch 获取处理器

根据 detected 类型获取 handler：

```cpp
handler = registry::global().create(detected);
```

Handler 类型：
- HttpHandler
- Socks5Handler
- TrojanHandler
- VlessHandler
- ShadowsocksHandler
- UnknownHandler

详见 [[core/agent/dispatch/table|处理器分发表]]

### Step 6: Pipeline 处理

handler 调用 pipeline 处理协议：

```cpp
handler.handle(transport, config, router)
```

解析请求，建立上游连接。

详见 [[core/pipeline/overview|管道层总览]]

### Step 7: Router 建立上游

通过 router 建立上游连接：

```cpp
upstream = router.connect(target_address);
```

支持直连和代理转发。

详见 [[core/resolve/router|DNS 路由器]]

### Step 8: Tunnel 双向转发

透明转发客户端和上游数据：

```cpp
co_await tunnel(client_transport, upstream_transport);
```

双向读写，直到连接关闭。

详见 [[core/pipeline/primitives|管道原语]]

## 协议类型判断

| 首字节 | 协议类型 |
|--------|----------|
| 0x05 | SOCKS5 |
| 'G', 'P', 'C' | HTTP |
| 0x16 | TLS（需进一步识别） |
| 其他 | SS2022 fallback |

## Recognition 插件架构

新伪装方案只需：

1. 实现 `feature_analyzer` 子类
2. 注册：`REGISTER_CLIENTHELLO_ANALYZER()`

详见 [[dev/extending/stealth|新伪装方案开发]]