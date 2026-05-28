---
tags: [recognition, probe, analyzer]
layer: core
module: recognition
source: I:/code/Prism/include/prism/recognition/probe/analyzer.hpp
title: analyzer.hpp
---

# analyzer.hpp

外层协议检测（纯内存操作），通过魔术字节判断协议类型。包含 `detect()` 和 `detect_tls()` 两个检测函数。

## 核心函数

### detect()

```cpp
[[nodiscard]] auto detect(std::string_view peek_data) -> protocol::protocol_type;
```

从预读数据检测外层协议类型。检测顺序：SOCKS5 → TLS → HTTP → Shadowsocks（排除法 fallback）。空数据返回 `unknown`。

纯函数，无状态，线程安全，零堆分配。

### detect_tls()

```cpp
[[nodiscard]] inline auto detect_tls(std::string_view peek_data) -> protocol::protocol_type;
```

TLS 握手完成后探测内层协议类型（HTTP/VLESS/Trojan）。检测顺序：HTTP → VLESS → Trojan → unknown。

数据不足 60 字节且不匹配 HTTP/VLESS 时返回 `unknown`，让调用者继续读取。60+ 字节仍无法识别也返回 `unknown`。

### is_http_request()

```cpp
[[nodiscard]] inline auto is_http_request(std::string_view data) noexcept -> bool;
```

检查数据是否以已知 HTTP 方法前缀开头。匹配 `GET`、`POST`、`HEAD`、`PUT`、`DELETE`、`CONNECT`、`OPTIONS`、`TRACE`、`PATCH` 共 9 种方法。

## 设计决策

### 为什么 TLS 检测必须检查两字节？

SS2022 的 salt 是随机字节，首字节恰好为 `0x16` 的概率约 1/256。如果仅检查首字节，约 0.4% 的 SS2022 连接会被误判为 TLS。检查前两字节 `0x16 0x03` 将误判率降至 1/65536。

**后果**: SOCKS5 仅检查首字节（`0x05`），理论上仍有约 0.4% 误判率，但 SOCKS5 握手有后续验证阶段可纠正。

### 为什么 Shadowsocks 用排除法 fallback？

Shadowsocks 2022 数据包以随机 salt 开头，没有固定魔术字节。无法正向识别，只能排除所有已知协议后假定是 SS2022。

**后果**: 任何无法识别的流量都会被当作 Shadowsocks。如果将来新增无特征协议，必须在此处调整检测逻辑。

### 为什么 detect_tls() 需要至少 60 字节？

Trojan 协议头部最少 60 字节：56 字节 hex 编码密码 + `\r\n` + 1 字节命令 + 1 字节地址类型。不足 60 字节无法可靠区分 Trojan 和其他协议，返回 `unknown` 让调用者继续读取。

**后果**: 调用方需要处理 `unknown` 结果，可能需要读取更多数据后重试。

### 为什么 detect_tls() 是 inline 而 detect() 不是？

`detect_tls()` 逻辑较长但全部是 constexpr 兼容的字节比较，编译器可内联优化。`detect()` 调用 `is_http_request()`（遍历 9 种方法），分离编译避免调用站点的代码膨胀。

**后果**: `detect_tls()` 在 header 中定义，包含在多个编译单元不会增加二进制体积。

## 约束

### 探测结果非终局

**类型**: 状态前置

**规则**: `detect()` 和 `detect_tls()` 的结果基于有限数据，后续数据可能推翻当前判断。例如，误判为 SS2022 的连接在握手阶段会因解密失败而被纠正。

**违反后果**: 调用方不能假设检测结果 100% 正确，必须为后续阶段的失败准备回退路径。

**源码依据**: `analyzer.hpp:11-12`

### detect_tls() 数据来源要求

**类型**: 数据格式

**规则**: `detect_tls()` 的输入必须是 TLS 握手完成后读取的应用层数据，不是原始 TLS 记录。

**违反后果**: 如果传入 TLS 记录头（`0x17 0x03 0x03`），所有检测逻辑都会失败。

**源码依据**: `analyzer.hpp:62-64`

## 引用关系

### 依赖

- [[core/protocol/analysis|protocol::protocol_type]]：协议类型枚举

### 被引用

- [[core/recognition/probe/probe|probe]]：probe() 中调用 detect()
- [[core/stealth/scheme|stealth::scheme]]：伪装方案可能调用 detect_tls() 识别内层协议
