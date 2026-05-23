---
title: 启动流程详解
created: 2026-05-17
layer: core
tags: [architecture, startup]
source: CLAUDE.md §启动流程, main.cpp
---

# 启动流程详解

`main.cpp` 中的启动顺序，共 9 步。

## 启动顺序

### Step 1: 初始化全局内存池

```cpp
memory::system::enable_global_pooling()
```

初始化 PMR 全局内存池，所有热路径容器使用此池分配。

详见 [[core/memory/pool|内存池]]

### Step 2: 注册 TLS 伪装方案

```cpp
stealth::register_all_schemes()
```

注册所有伪装方案到 registry：
- Reality
- ShadowTLS
- Restls
- AnyTLS
- TrustTunnel
- Native TLS

详见 [[core/stealth/registry|方案注册表]]

### Step 3: 加载配置文件

```cpp
loader::load(config_path)
```

配置文件查找顺序：
1. 命令行参数指定的路径
2. 可执行文件同目录下的 `configuration.json`

详见 [[core/loader/load|配置加载]]

### Step 4: 初始化日志系统

```cpp
trace::init(trace_config)
```

配置 spdlog：
- 文件日志（轮转）
- 控制台日志
- 日志级别

详见 [[core/trace/config|日志配置]]

### Step 5: 构建账户目录

```cpp
loader::build_account_directory(config)
```

从配置的 `authentication.users` 构建账户目录，用于后续认证。

详见 [[core/instance/account/directory|账户目录]]

### Step 6: 创建 Worker 线程池

```cpp
// CPU 核心数 - 1 个 worker
workers.resize(std::thread::hardware_concurrency() - 1)
```

每个 worker 持有独立的 `io_context`，处理连接。

详见 [[core/instance/worker/worker|Worker]]

### Step 7: 构建负载均衡器

```cpp
balancer{workers}
```

负载均衡器持有 worker 引用，根据亲和性哈希分发连接。

详见 [[core/instance/front/balancer|负载均衡器]]

### Step 8: 启动监听器

```cpp
listener{balancer, endpoint, tls_context}
```

监听配置的端点，接受连接后分发给 balancer。

详见 [[core/instance/front/listener|监听器]]

### Step 9: 启动所有线程

```cpp
// 启动 worker 线程
for (auto& w : workers) w.start();

// 启动监听线程
listener_thread = std::thread{[&] { listener.run(); }};
```

所有线程开始运行，处理连接。

## 配置结构

配置文件 `configuration.json` 主要部分：

```json
{
  "agent": {
    "addressable": { "host": "0.0.0.0", "port": 8081 },
    "certificate": { "key": "...", "cert": "..." },
    "authentication": { "users": [...] }
  },
  "pool": { "max_cache_per_endpoint": 32, ... },
  "protocol": { "socks5": {...}, "trojan": {...}, ... },
  "multiplex": { "smux": {...}, "yamux": {...} },
  "stealth": { "reality": {...}, "shadowtls": {...}, ... },
  "dns": { "servers": [...], ... },
  "trace": { "log_level": "debug", ... }
}
```

详见 [[docs/configuration|配置说明]]

## 启动流程图

```
┌─────────────────────────────────────────────────────────────┐
│  main.cpp                                                   │
├─────────────────────────────────────────────────────────────┤
│  1. memory::system::enable_global_pooling()                 │
│     ↓                                                       │
│  2. stealth::register_all_schemes()                         │
│     ↓                                                       │
│  3. loader::load(config_path)                               │
│     ↓                                                       │
│  4. trace::init(config.trace)                               │
│     ↓                                                       │
│  5. loader::build_account_directory(config)                 │
│     ↓                                                       │
│  6. 创建 workers (CPU核心数-1)                               │
│     ↓                                                       │
│  7. 构建 balancer                                           │
│     ↓                                                       │
│  8. 构建 listener                                           │
│     ↓                                                       │
│  9. 启动所有线程                                             │
└─────────────────────────────────────────────────────────────┘
```

## 注意事项

- 配置文件必须存在，否则启动失败
- TLS 证书路径必须正确
- Worker 数量至少 1 个
- 监听端口不能被占用