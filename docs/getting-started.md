---
title: 快速开始
created: 2026-05-17
updated: 2026-05-17
layer: docs
tags: [docs, guide]
---

# 快速开始

5 分钟上手 Prism，从编译到测试连接。

## 功能简介

提供从源码编译、预编译二进制获取、最小配置到启动验证的完整快速入门流程。

## 环境要求

- **编译器**: GCC >= 13 或 Clang >= 16（必须支持 C++23 协程）
- **构建工具**: CMake >= 3.23
- **操作系统**: Windows 10+ / Linux (glibc 2.31+)
- **可选依赖**: OpenSSL 3.x（用于 TLS 证书生成）

## 安装

### 源码编译（推荐）

```bash
# 克隆仓库
git clone https://github.com/Hatedatastructures/Prism.git
cd Prism

# Debug 版本（含符号信息，便于调试）
cmake -B build_debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build_debug --config Debug

# Release 版本（生产环境使用）
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release
```

编译成功后，二进制文件位于 `build_release/src/Prism`（Linux）或 `build_release/src/Prism.exe`（Windows）。

### 预编译二进制

从 [GitHub Releases](https://github.com/Hatedatastructures/Prism/releases) 下载对应平台的预编译包：

```bash
# Linux
tar -xzf prism-linux-x64.tar.gz
sudo mv prism /usr/local/bin/

# Windows
# 解压 prism-windows-x64.zip 到 C:\Prism\
```

> **注意**: 预编译包默认包含动态链接库，需确保系统满足运行依赖要求。

## 快速配置

在 Prism 工作目录创建 `configuration.json`，以下是最小可用配置：

```json
{
  "agent": {
    "addressable": {
      "host": "0.0.0.0",
      "port": 8081
    }
  },
  "pool": {
    "max_cache_per_endpoint": 32,
    "max_idle_seconds": 60
  },
  "trace": {
    "log_level": "info"
  }
}
```

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `agent.addressable.host` | 监听地址，`0.0.0.0` 接受所有网卡连接 | `127.0.0.1` |
| `agent.addressable.port` | 监听端口 | `8080` |
| `pool.max_cache_per_endpoint` | 每个目标地址的最大缓存连接数 | `32` |
| `pool.max_idle_seconds` | 空闲连接超时时间（秒） | `60` |
| `trace.log_level` | 日志级别：`trace`/`debug`/`info`/`warn`/`error` | `info` |

更多配置项详见 [[docs/configuration|配置说明]]。

## 启动与验证

### 启动服务

```bash
# 使用默认路径 configuration.json
./build_release/src/Prism

# 指定配置文件路径
./build_release/src/Prism --config /etc/prism/configuration.json
```

### 验证服务

1. **端口检查**: `netstat -tlnp | grep 8081` 确认端口已监听
2. **连通性测试**: `telnet 127.0.0.1 8081` 或 `nc -zv 127.0.0.1 8081`
3. **HTTP 代理测试**:
   ```bash
   curl -x http://127.0.0.1:8081 http://example.com -v
   ```
4. **日志检查**: 查看日志输出，确认无 `error` 级别信息

### 健康检查脚本

```bash
#!/bin/bash
PORT=8081
if nc -z 127.0.0.1 $PORT 2>/dev/null; then
  echo "Prism is running on port $PORT"
else
  echo "Prism is NOT running on port $PORT"
  exit 1
fi
```

## 下一步

- [[docs/configuration|配置说明]]: 了解全部配置项及其作用
- [[docs/client-setup|客户端配置]]: 配置 mihomo/Clash 客户端连接
- [[docs/deployment|部署指南]]: 生产环境部署（systemd / Docker）
- [[docs/security|安全注意事项]]: 安全加固建议

## 常见问题

**Q: 编译失败怎么办？**
A: 确认 GCC 版本 >= 13，CMake >= 3.23。使用 `gcc --version` 和 `cmake --version` 检查。如需 C++23 协程支持，编译器必须足够新。

**Q: 端口已被占用？**
A: 修改 `agent.addressable.port` 为其他值，或使用 `lsof -i :8081`（Linux）/ `netstat -ano | findstr :8081`（Windows）查找占用进程。

**Q: 如何测试代理？**
A: `curl -x http://127.0.0.1:8081 http://example.com`。如果返回 HTTP 200 和页面内容，说明代理工作正常。

**Q: 启动后闪退？**
A: 使用 `./Prism --config config.json 2>&1 | tee prism.log` 捕获输出，检查日志中的 `error` 信息。

## 相关页面

- [[docs/deployment|部署指南]]
- [[docs/configuration|配置说明]]
- [[docs/troubleshooting|故障排查]]
- [[docs/faq|常见问题]]
