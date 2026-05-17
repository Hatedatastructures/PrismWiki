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

提供从源码编译、配置到启动测试的完整快速入门流程。

## 如何启用

```bash
# 克隆仓库
git clone https://github.com/Hatedatastructures/Prism.git
cd Prism

# 编译 Release 版本
cmake -B build_release -DCMAKE_BUILD_TYPE=Release
cmake --build build_release --config Release

# 启动
./build_release/src/Prism
```

最简配置（`configuration.json`）:

```json
{
  "agent": {
    "addressable": {
      "host": "0.0.0.0",
      "port": 8081
    }
  }
}
```

## 常见问题

**Q: 编译失败怎么办？**
A: 确认 GCC 版本 >= 13，CMake >= 3.23。

**Q: 如何测试代理？**
A: `curl -x http://127.0.0.1:8081 http://example.com`

## 相关页面

- [[docs/deployment|部署指南]]
- [[docs/configuration|配置说明]]
- [[docs/troubleshooting|故障排查]]