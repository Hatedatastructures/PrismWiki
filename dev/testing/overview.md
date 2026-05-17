---
layer: dev
source: I:/code/Prism/tests/
module: testing
type: overview
tags: [testing, unit-test, integration, e2e, ctest]
created: 2026-05-17
updated: 2026-05-17
related:
  - "[[dev/testing/framework]]"
  - "[[dev/testing/commands]]"
  - "[[dev/testing/benchmark]]"
  - "[[dev/testing/stress]]"
  - "[[dev/testing/concurrency]]"
---

# 测试体系概述

Prism 采用独立可执行文件测试架构，包含 44 个测试覆盖基础设施、协议、加密、多路复用、伪装、识别、管道、DNS、端到端和回归等维度。

## 测试架构设计

```
+--------------------------------------------------+
|                    测试入口                       |
|              ctest --test-dir build              |
+--------------------------------------------------+
                       |
      +----------------+----------------+
      |                |                |
+----------+    +------------+    +------------+
| 单元测试 |    | 集成测试   |    | E2E测试   |
+----------+    +------------+    +------------+
      |                |                |
      +----+----+      +----+----+      +----+
           |                |                |
    +------+------+  +------+------+  +------+------+
    |基础设施测试|  |协议集成测试|  |并发测试    |
    +------+------+  +------+------+  +------+------+
           |                |                |
    +------+------+  +------+------+        |
    |加密模块测试|  |伪装模块测试|        |
    +------+------+  +------+------+        |
           |                |                |
    +------+------+  +------+------+        |
    |多路复用测试|  |识别模块测试|        |
    +------+------+  +------+------+        |
           |                                     |
    +------+------+                        +----+
    |DNS模块测试 |                        |回归|
    +------+------+                        +----+
```

每个测试是独立可执行文件的优势：

| 优势 | 说明 |
|------|------|
| 隔离性 | 测试失败不影响其他测试 |
| 并行性 | 多个测试可同时运行 |
| 调试性 | 单独运行便于问题定位 |
| 可定制 | 每个测试可独立配置 |

## 测试分类总览

Prism 的 44 个测试按模块分类：

| 分类 | 测试数 | 测试列表 |
|------|--------|----------|
| 基础设施 | 4 | Session, Connection, Transmission, Trace |
| 协议 | 5 | Http, Socks5, Trojan, Vless, Shadowsocks |
| 加密 | 6 | Crypto, Aead, Blake3, Hkdf, X25519, Block |
| 多路复用 | 5 | Smux, SmuxCraft, Yamux, YamuxCraft, MultiplexDuct |
| 伪装 | 5 | Reality, Shadowtls, ShadowTlsE2E, Restls, Executor |
| 识别 | 4 | Recognition, FeatureBitmap, ProtocolAnalysis, SchemeRouteTable |
| 管道 | 1 | PipelinePrimitives |
| DNS | 3 | DnsCache, DnsPacket, DnsRules |
| 基础 | 8 | Fault, Exception, MemoryArena, Config, Balancer, AccountDirectory, Base64Encode, FaultHandling |
| HTTP解析 | 1 | HttpParser |
| 协议工具 | 1 | ProtocolToString |
| 端到端 | 1 | E2E |
| 回归 | 1 | Regression |

## 测试文件位置

```
tests/
├── common/
│   └── TestRunner.hpp          # 测试框架
│   └── MockTlsServer.hpp       # TLS Mock
├── concurrency/
│   ├── server.cpp              # 并发测试服务器
│   ├── client.cpp              # 并发测试客户端
│   ├── conversation.hpp        # 会话定义
│   ├── handler.hpp             # 处理器
│   └── server.hpp              # 服务器声明
├── Session.cpp                 # 会话测试
├── Connection.cpp              # 连接池测试
├── Transmission.cpp            # 传输测试
├── Trace.cpp                   # 日志测试
├── Http.cpp                    # HTTP 协议测试
├── HttpParser.cpp              # HTTP 解析器测试
├── Socks5.cpp                  # SOCKS5 协议测试
├── Trojan.cpp                  # Trojan 协议测试
├── Vless.cpp                   # VLESS 协议测试
├── Shadowsocks.cpp             # SS2022 协议测试
├── Crypto.cpp                  # 加密模块测试
├── Aead.cpp                    # AEAD 加密测试
├── Blake3.cpp                  # BLAKE3 哈希测试
├── Hkdf.cpp                    # HKDF 密钥派生测试
├── X25519.cpp                  # X25519 密钥交换测试
├── Block.cpp                   # 块加密测试
├── Base64Encode.cpp            # Base64 编码测试
├── Smux.cpp                    # smux 帧解析测试
├── SmuxCraft.cpp               # smux 帧构建测试
├── Yamux.cpp                   # yamux 帧解析测试
├── YamuxCraft.cpp              # yamux 帧构建测试
├── MultiplexDuct.cpp           # 多路复用管道测试
├── Reality.cpp                 # Reality TLS 测试
├── Shadowtls.cpp               # ShadowTLS 测试
├── ShadowTlsE2E.cpp            # ShadowTLS E2E 测试
├── Restls.cpp                  # Restls 测试
├── Executor.cpp                # 执行器测试
├── Recognition.cpp             # 协议识别测试
├── FeatureBitmap.cpp           # 特征位图测试
├── ProtocolAnalysis.cpp        # 协议分析测试
├── SchemeRouteTable.cpp        # 方案路由表测试
├── PipelinePrimitives.cpp      # 管道原语测试
├── DnsCache.cpp                # DNS 缓存测试
├── DnsPacket.cpp               # DNS 报文测试
├── DnsRules.cpp                # DNS 规则测试
├── Fault.cpp                   # 错误码测试
├── Exception.cpp               # 异常测试
├── MemoryArena.cpp             # 帧竞技场测试
├── Config.cpp                  # 配置解析测试
├── Balancer.cpp                # 负载均衡测试
├── AccountDirectory.cpp        # 账户目录测试
├── FaultHandling.cpp           # 错误处理测试
├── ProtocolToString.cpp        # 协议字符串转换测试
├── E2E.cpp                     # 端到端测试
├── Regression.cpp              # 回归测试
└── CMakeLists.txt              # 测试构建配置
```

## 运行方式

```bash
# 运行全部测试（CTest）
ctest --test-dir build_release --output-on-failure

# 运行单个测试
build_release/tests/Session.exe
build_release/tests/Socks5.exe
build_release/tests/Crypto.exe

# 并行运行测试（4 线程）
ctest --test-dir build_release -j 4
```

## 构建配置

测试在 `tests/CMakeLists.txt` 中通过 `forward_add_test()` 函数统一添加：

```cmake
function(forward_add_test test_name source_file)
    add_executable(${test_name} ${CMAKE_CURRENT_SOURCE_DIR}/${source_file})
    target_compile_options(${test_name} PRIVATE ${TEST_COMPILE_OPTS})
    target_link_libraries(${test_name} PRIVATE ${PROJECT_NAME}_static_library)
    if(MINGW)
        target_link_options(${test_name} PRIVATE -static -static-libgcc -static-libstdc++)
    endif()
    add_test(NAME ${test_name} COMMAND ${test_name})
endfunction()
```

编译选项：
- `-g1` — 最小调试信息
- `-Os` — 优化大小
- MinGW: `-Wa,-mbig-obj` 避免大目标文件问题

## 相关链接

- [[dev/testing/framework]] — TestRunner 测试框架
- [[dev/testing/writing]] — 测试编写指南
- [[dev/testing/commands]] — 测试命令详解
- [[dev/testing/benchmark]] — 基准测试体系
- [[dev/testing/stress]] — 压力测试体系
- [[dev/testing/concurrency]] — 并发测试体系