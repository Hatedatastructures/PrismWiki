---
title: "format.hpp — DNS 报文编解码"
source: "include/prism/resolve/dns/detail/format.hpp"
module: "resolve"
type: api
tags: [resolve, dns, format, DNS报文, RFC1035, 压缩指针]
created: 2026-05-15
updated: 2026-05-15
related:
  - resolve/dns/upstream
  - resolve/dns/detail/cache
  - resolve/dns/dns
  - memory/container
---

# format.hpp

> 源码: `include/prism/resolve/dns/detail/format.hpp` + `src/prism/resolve/dns/detail/format.cpp`
> 模块: [[resolve|Resolve]] / dns / detail

## 概述

DNS 二进制报文的构造与解析（RFC 1035），完全不依赖系统 resolver。支持域名压缩指针编解码、多种记录类型（A/AAAA/CNAME/NS/MX/TXT/SOA/PTR/OPT），以及 TCP 帧格式的封装与解析。

实现细节：
- 域名编码采用压缩指针优化，从最长后缀开始查找压缩机会
- 域名解码支持压缩指针递归跳转，循环检测阈值为 255 次
- 所有字符串与容器均通过 PMR 分配器管理
- 预分配 512 字节缓冲区容量（DNS 传统最大报文长度）

## 依赖关系

| 依赖方向 | 模块 | 说明 |
|----------|------|------|
| 依赖 | [[memory/container|container]] | PMR 容器类型 |
| 被依赖 | [[resolve/dns/upstream|upstream]] | 上游查询使用报文编解码 |
| 被依赖 | [[resolve/dns/detail/cache|cache]] | 缓存使用 qtype 枚举 |
| 被依赖 | [[resolve/dns/dns|dns]] | resolver 实现使用报文类型 |

## 命名空间

`psm::resolve::dns::detail`

---

## 枚举: qtype

- **功能说明**: DNS 查询/资源记录类型枚举，对应 RFC 1035 定义的 QTYPE / TYPE 字段，仅列出本项目实际使用到的类型。
- **签名**:
  ```cpp
  enum class qtype : std::uint16_t {
      a = 1,      // IPv4 地址记录
      ns = 2,     // 权威名称服务器
      cname = 5,  // 规范名称（别名）
      soa = 6,    // 区域起始授权
      mx = 15,    // 邮件交换
      txt = 16,   // 文本记录
      aaaa = 28,  // IPv6 地址记录
      opt = 41,   // EDNS0 选项
  };
  ```
- **参数**: 无（枚举）
- **返回值**: 无（枚举类型）
- **调用（向下）**: 无
- **被调用（向上）**: [[resolve/dns/upstream|upstream]]、[[resolve/dns/detail/cache|cache]]、[[resolve/dns/dns|resolver]] 广泛使用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]

---

## 结构体: question

### 概述

DNS 查询段（Question Section）。每个查询段包含一个域名、查询类型和查询类。域名存储为小写、无末尾点号的 dotted 格式（如 `"www.example.com"`）。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `name` | 域名，小写无末尾点号 |
| `qtype` | `query_type` | 查询类型 |
| `std::uint16_t` | `qclass` | 查询类，默认 IN (1) |

---

### 函数: question::question()

- **功能说明**: 构造 DNS 查询段，使用指定内存资源初始化域名成员。
- **签名**:
  ```cpp
  explicit question(memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | PMR 内存资源指针 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `message::make_query()` 和 `message::unpack()` 内部构造
- **知识域**: [[memory/container|PMR 容器]]

---

## 结构体: record

### 概述

DNS 资源记录（Resource Record）。每条记录包含拥有者名称、类型、类、TTL 及原始 RDATA。rdata 保存未解码的二进制数据，可通过 `extract_ipv4()` / `extract_ipv6()` 等工具函数提取具体语义。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `memory::string` | `name` | 拥有者名称，小写无末尾点号 |
| `qtype` | `type` | 记录类型 |
| `std::uint16_t` | `rclass` | 记录类，默认 IN |
| `std::uint32_t` | `ttl` | 生存时间（秒） |
| `memory::vector<std::uint8_t>` | `rdata` | 原始 RDATA |

---

### 函数: record::record()

- **功能说明**: 构造 DNS 资源记录，使用指定内存资源初始化名称和 RDATA 容器成员。
- **签名**:
  ```cpp
  explicit record(memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | PMR 内存资源指针 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `message::unpack()` 内部构造
- **知识域**: [[memory/container|PMR 容器]]

---

## 自由函数: extract_ipv4()

- **功能说明**: 从 A 记录中提取 IPv4 地址。检查记录类型是否为 `qtype::a` 且 rdata 长度是否恰好为 4 字节，若是则按大端序构造 `address_v4` 对象。
- **签名**:
  ```cpp
  [[nodiscard]] auto extract_ipv4(const record &rec) -> std::optional<net::ip::address_v4>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `rec` | `const record &` | DNS 资源记录 |

- **返回值**: `std::optional<net::ip::address_v4>` — 若 rdata 恰好 4 字节则返回对应地址，否则返回 `std::nullopt`
- **调用（向下）**: 无（直接从 rdata 字节构造地址）
- **被调用（向上）**: `message::extract_ips()` 内部遍历调用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]

---

## 自由函数: extract_ipv6()

- **功能说明**: 从 AAAA 记录中提取 IPv6 地址。检查记录类型是否为 `qtype::aaaa` 且 rdata 长度是否恰好为 16 字节，若是则构造 `address_v6` 对象。
- **签名**:
  ```cpp
  [[nodiscard]] auto extract_ipv6(const record &rec) -> std::optional<net::ip::address_v6>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `rec` | `const record &` | DNS 资源记录 |

- **返回值**: `std::optional<net::ip::address_v6>` — 若 rdata 恰好 16 字节则返回对应地址，否则返回 `std::nullopt`
- **调用（向下）**: 无（直接从 rdata 字节构造地址）
- **被调用（向上）**: `message::extract_ips()` 内部遍历调用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]

---

## 类: message

### 概述

DNS 报文（RFC 1035），表示一条完整的 DNS 报文，包含 Header、Question、Answer、Authority、Additional 四个段。提供序列化（`pack`）与反序列化（`unpack`）能力，支持域名压缩指针编解码。

### 成员变量

| 类型 | 名称 | 说明 |
|------|------|------|
| `std::uint16_t` | `id` | 报文标识 |
| `bool` | `qr` | 0=查询, 1=响应 |
| `std::uint8_t` | `opcode` | 操作码（0=标准查询） |
| `bool` | `aa` | 权威应答 |
| `bool` | `tc` | 截断标志 |
| `bool` | `rd` | 期望递归 |
| `bool` | `ra` | 可用递归 |
| `std::uint8_t` | `rcode` | 响应码 |
| `memory::vector<question>` | `questions` | 查询段 |
| `memory::vector<record>` | `answers` | 应答段 |
| `memory::vector<record>` | `authority` | 权威段 |
| `memory::vector<record>` | `additional` | 附加段 |

---

### 函数: message::message()

- **功能说明**: 构造 DNS 报文，使用指定内存资源初始化所有 PMR 容器成员。
- **签名**:
  ```cpp
  explicit message(memory::resource_pointer mr = memory::current_resource());
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `mr` | `memory::resource_pointer` | PMR 内存资源指针 |

- **返回值**: 无（构造函数）
- **调用（向下）**: 无
- **被调用（向上）**: `make_query()`、`unpack()` 以及 [[resolve/dns/upstream|upstream]] 内部构造
- **知识域**: [[memory/container|PMR 容器]]

---

### 函数: message::pack()

- **功能说明**: 序列化为 DNS wire format。按照 12 字节 Header → Question 列表 → Answer/Authority/Additional 记录列表的顺序编码。域名编码采用压缩指针优化，预分配 512 字节缓冲区容量。
- **签名**:
  ```cpp
  [[nodiscard]] auto pack() const -> memory::vector<std::uint8_t>;
  ```
- **参数**: 无
- **返回值**: `memory::vector<std::uint8_t>` — 完整的二进制报文字节序列
- **调用（向下）**: `encode_name()` → 内部压缩指针编码
- **被调用（向上）**: [[resolve/dns/upstream|upstream]] 的 `query_via()` 模板函数在发送前调用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]、域名压缩指针

---

### 函数: message::unpack() [static]

- **功能说明**: 从二进制数据反序列化 DNS 报文。检查数据长度 >= 12 字节，解析 Header 后逐段解码 Question、Answer、Authority、Additional 记录。域名解码时处理压缩指针并检测循环引用（跳转 > 255 次视为循环）。
- **签名**:
  ```cpp
  [[nodiscard]] static auto unpack(std::span<const std::uint8_t> data,
                                   memory::resource_pointer mr = memory::current_resource())
      -> std::optional<message>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `data` | `std::span<const std::uint8_t>` | 包含 DNS 报文的字节缓冲区 |
  | `mr` | `memory::resource_pointer` | PMR 内存资源指针 |

- **返回值**: `std::optional<message>` — 解析成功返回 message，否则返回 `std::nullopt`
- **调用（向下）**: `read_u16_be()` / `read_u32_be()` → `decode_name()` → `parse_records()`
- **被调用（向上）**: [[resolve/dns/upstream|upstream]] 的 `query_via()` 在接收响应后调用；`unpack_tcp()` 委托调用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]、域名压缩指针

---

### 函数: message::make_query() [static]

- **功能说明**: 创建标准递归查询报文。设置 `id=0`、`rd=true`、`opcode=0`，添加一个 Question 段。域名自动转为小写并去除末尾点号。
- **签名**:
  ```cpp
  [[nodiscard]] static auto make_query(std::string_view domain, qtype qt,
                                       memory::resource_pointer mr = memory::current_resource())
      -> message;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `domain` | `std::string_view` | 待查询域名 |
  | `qt` | `qtype` | 查询类型 |
  | `mr` | `memory::resource_pointer` | PMR 内存资源指针 |

- **返回值**: `message` — 构造好的查询报文
- **调用（向下）**: `to_lower_domain()` → 构造 `question` → 添加到 `questions`
- **被调用（向上）**: [[resolve/dns/upstream|upstream::resolve()]] 在构造查询报文时调用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]

---

### 函数: message::extract_ips()

- **功能说明**: 提取所有 A/AAAA 记录的 IP 地址。遍历应答段、权威段和附加段中的所有记录，将 A 记录映射为 v4 地址、AAAA 记录映射为 v6 地址，跳过其他类型。
- **签名**:
  ```cpp
  [[nodiscard]] auto extract_ips() const -> memory::vector<net::ip::address>;
  ```
- **参数**: 无
- **返回值**: `memory::vector<net::ip::address>` — 包含所有有效 IP 地址的列表
- **调用（向下）**: `extract_ipv4()` + `extract_ipv6()` 遍历各段记录
- **被调用（向上）**: [[resolve/dns/upstream|upstream]] 的 `query_via()` 在成功解析响应后调用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]

---

### 函数: message::min_ttl()

- **功能说明**: 计算所有记录中的最小 TTL。遍历应答段、权威段和附加段中的所有记录，取最小 TTL 值。若无任何记录则返回 0。
- **签名**:
  ```cpp
  [[nodiscard]] auto min_ttl() const -> std::uint32_t;
  ```
- **参数**: 无
- **返回值**: `std::uint32_t` — 所有段中记录的最小 TTL 值；若无任何记录则返回 0
- **调用（向下）**: 遍历 `answers`、`authority`、`additional` 取最小值
- **被调用（向上）**: [[resolve/dns/dns|resolver_impl::query_pipeline()]] 在 TTL 钳制阶段调用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]

---

## 自由函数: pack_tcp()

- **功能说明**: 将 DNS 报文封装为 TCP 帧格式，在 DNS 报文前添加 2 字节大端长度前缀。（声明于头文件，实现待补全）
- **签名**:
  ```cpp
  [[nodiscard]] auto pack_tcp(const message &msg) -> memory::vector<std::uint8_t>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `msg` | `const message &` | 待封装的 DNS 报文 |

- **返回值**: `memory::vector<std::uint8_t>` — 包含长度前缀和报文字节的字节序列
- **调用（向下）**: `msg.pack()` → 添加 2 字节长度前缀
- **被调用（向上）**: [[resolve/dns/upstream|upstream]] TCP/TLS 传输中使用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]

---

## 自由函数: unpack_tcp()

- **功能说明**: 从 TCP 帧中解析 DNS 报文。前 2 字节为大端长度前缀，其后为 DNS 报文主体。检查数据完整性后委托给 `message::unpack()` 解析。
- **签名**:
  ```cpp
  [[nodiscard]] auto unpack_tcp(std::span<const std::uint8_t> data,
                                memory::resource_pointer mr = memory::current_resource())
      -> std::optional<message>;
  ```
- **参数**:

  | 参数 | 类型 | 说明 |
  |------|------|------|
  | `data` | `std::span<const std::uint8_t>` | 包含 TCP 帧的完整字节缓冲区 |
  | `mr` | `memory::resource_pointer` | PMR 内存资源指针 |

- **返回值**: `std::optional<message>` — 解析成功返回 message，否则返回 `std::nullopt`
- **调用（向下）**: 读取 2 字节长度 → `message::unpack(data.subspan(2, length), mr)`
- **被调用（向上）**: [[resolve/dns/upstream|upstream]] TCP/TLS 传输中接收响应后调用
- **知识域**: [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]

---

## 内部实现细节

### 域名编码 (`encode_name`)

从最长后缀开始查找压缩机会。例如 `"www.example.com"` 依次尝试：
1. `"www.example.com"` → 未命中，写入标签 `"www"`，登记偏移
2. `"example.com"` → 未命中，写入标签 `"example"`，登记偏移
3. `"com"` → 未命中，写入标签 `"com"`，登记偏移

若某后缀命中则写入 2 字节压缩指针（`0xC000 | offset`），跳过后续标签。

### 域名解码 (`decode_name_raw`)

使用栈上 256 字节缓冲区拼接域名标签，避免中间分配。处理逻辑：
- `< 0xC0`：普通标签，读长度 + 内容
- `>= 0xC0`：压缩指针，低 14 位为偏移量，递归跳转
- 跳转计数 > 255 视为循环，返回空 `string_view`

---

## 调用链总览

```
[[resolve/dns/upstream|upstream::query_via()]]
  ├── message::make_query() → pack()   → encode_name() → 发送
  └── message::unpack()                 → decode_name() → parse_records()
      └── extract_ips() → extract_ipv4() / extract_ipv6()

pack_tcp()   → message::pack() + 2B 长度前缀
unpack_tcp() → 读取 2B 长度 → message::unpack()
```

---

## 知识域

- [[ref/protocol/dns-over-udp|DNS 协议 (RFC 1035)]]
- [[resolve/dns/detail/format|域名压缩指针]]
- [[ref/memory/pmr|PMR 内存管理]]
