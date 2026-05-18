---
title: "message — DNS 报文编解码"
layer: core
source: "I:/code/Prism/include/prism/resolve/dns/detail/format.hpp"
module: "resolve/dns/detail"
type: component
tags: [dns, format, wire, encode, decode, rfc1035]
created: 2026-05-17
updated: 2026-05-17
related:
  - core/resolve/dns/upstream
  - core/resolve/dns/detail/cache
---

# message — DNS 报文编解码

> 源码位置: `I:/code/Prism/include/prism/resolve/dns/detail/format.hpp`
> 模块: [[core/resolve|resolve]] / [[core/resolve/dns|dns]] / detail

## 组件定位

该文件实现 DNS 二进制报文的构造与解析（RFC 1035），完全不依赖系统 resolver。支持域名压缩指针、多种记录类型，以及 TCP 帧格式的封装。

## qtype 枚举

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

仅列出本项目实际使用的类型。

## question 结构

```cpp
struct question {
    memory::string name;      // 域名，小写无末尾点号
    qtype query_type{};       // 查询类型
    std::uint16_t qclass{1};  // 查询类，默认 IN（Internet）
};
```

表示 DNS 查询段（Question Section）中的一个条目。

## record 结构

```cpp
struct record {
    memory::string name;                 // 拥有者名称
    qtype type{};                        // 记录类型
    std::uint16_t rclass{1};             // 记录类
    std::uint32_t ttl{0};                // 生存时间（秒）
    memory::vector<std::uint8_t> rdata;  // 原始 RDATA
};
```

表示 DNS 资源记录（Resource Record）。`rdata` 保存未解码的二进制数据。

## 辅助函数

### extract_ipv4

```cpp
auto extract_ipv4(const record &rec) -> std::optional<net::ip::address_v4>;
```

从 A 记录中提取 IPv4 地址：
- 检查 `rdata.size() == 4`
- 构造 `address_v4`

### extract_ipv6

```cpp
auto extract_ipv6(const record &rec) -> std::optional<net::ip::address_v6>;
```

从 AAAA 记录中提取 IPv6 地址：
- 检查 `rdata.size() == 16`
- 构造 `address_v6`

## message 类

### 报文结构

```cpp
class message {
public:
    // Header
    std::uint16_t id{0};    // 报文标识
    bool qr{false};         // 0=查询, 1=响应
    std::uint8_t opcode{0}; // 操作码（0=标准查询）
    bool aa{false};         // 权威应答
    bool tc{false};         // 截断标志
    bool rd{false};         // 期望递归
    bool ra{false};         // 可用递归
    std::uint8_t rcode{0};  // 响应码

    // Sections
    memory::vector<question> questions; // 查询段
    memory::vector<record> answers;     // 应答段
    memory::vector<record> authority;   // 权威段
    memory::vector<record> additional;  // 附加段
};
```

### 核心方法

#### pack — 序列化

```cpp
auto pack() const -> memory::vector<std::uint8_t>;
```

编码顺序：
1. **Header** — 12 字节固定头部
2. **Question Section** — 查询段列表
3. **Answer Section** — 应答段列表
4. **Authority Section** — 权威段列表
5. **Additional Section** — 附加段列表

**域名编码**: 采用压缩指针优化，重复域名使用指针引用。

#### unpack — 反序列化

```cpp
static auto unpack(std::span<const std::uint8_t> data,
                   memory::resource_pointer mr = memory::current_resource())
    -> std::optional<message>;
```

流程：
1. 检查 `data.size() >= 12`
2. 解析 Header
3. 逐段解码 Question/Answer/Authority/Additional
4. 处理域名压缩指针
5. 检测循环引用（跳转 > 255 次）

**错误处理**: 解析失败返回 `std::nullopt`。

#### make_query — 构造查询报文

```cpp
static auto make_query(std::string_view domain, qtype qt,
                       memory::resource_pointer mr = memory::current_resource())
    -> message;
```

设置：
- `id=0`（调用方在发送前自行设置）
- `rd=true`（期望递归）
- `opcode=0`（标准查询）
- 添加一个 Question

**域名规范化**: 自动转小写并去除末尾点号。

#### extract_ips — 提取 IP 地址

```cpp
auto extract_ips() const -> memory::vector<net::ip::address>;
```

遍历 `answers`：
- A 记录 → `extract_ipv4()` → IPv4 地址
- AAAA 记录 → `extract_ipv6()` → IPv6 地址
- 其他类型 → 跳过

#### min_ttl — 计算最小 TTL

```cpp
auto min_ttl() const -> std::uint32_t;
```

遍历 `answers`/`authority`/`additional`，返回最小 TTL 值。

## TCP 帧格式

### pack_tcp

```cpp
auto pack_tcp(const message &msg) -> memory::vector<std::uint8_t>;
```

在 DNS 报文前添加 2 字节大端长度前缀：

```
[2 bytes length (big-endian)] [DNS message bytes]
```

### unpack_tcp

```cpp
auto unpack_tcp(std::span<const std::uint8_t> data,
                memory::resource_pointer mr = memory::current_resource())
    -> std::optional<message>;
```

先读 2 字节长度，再解析报文主体。

## DNS Wire Format 详解

### Header 格式（12 字节）

```
+---------------------+
| ID (2 bytes)        |
+---------------------+
| FLAGS (2 bytes)     |  QR(1) OPCODE(4) AA(1) TC(1) RD(1) RA(1) Z(3) RCODE(4)
+---------------------+
| QDCOUNT (2 bytes)   |  Question 段条目数
+---------------------+
| ANCOUNT (2 bytes)   |  Answer 段条目数
+---------------------+
| NSCOUNT (2 bytes)   |  Authority 段条目数
+---------------------+
| ARCOUNT (2 bytes)   |  Additional 段条目数
+---------------------+
```

### 域名编码

**标签格式**:
```
[length byte (0-63)] [label bytes]
```

**压缩指针**:
```
[11xx xxxx] [offset byte]
```
- 高 2 位为 `11` 表示指针
- 后 14 位为报文内偏移位置

**示例**:
```
"www.example.com" 编码:
[3] "www" [7] "example" [3] "com" [0]  // 末尾 0 表示结束

若后续域名为 "api.example.com":
[3] "api" [pointer to "example.com"]   // 压缩指针复用
```

### Record 格式

```
+---------------------+
| NAME (compressed)   |
+---------------------+
| TYPE (2 bytes)      |
+---------------------+
| CLASS (2 bytes)     |
+---------------------+
| TTL (4 bytes)       |
+---------------------+
| RDLENGTH (2 bytes)  |
+---------------------+
| RDATA (RDLENGTH)    |
+---------------------+
```

## 使用示例

```cpp
// 构造查询
auto query = message::make_query("www.example.com", qtype::a);
query.id = 12345;  // 设置报文 ID

// 序列化
auto wire = query.pack();

// 发送到上游服务器...
// 接收响应...

// 解析响应
auto response = message::unpack(response_wire);
if (response) {
    auto ips = response->extract_ips();
    auto ttl = response->min_ttl();
}
```

## 调用链

```
upstream::query_udp(server, query)
  │
  ├─→ message::make_query(domain, qtype::a)
  │     → 设置 rd=true, opcode=0
  │     → 添加 question
  │
  ├─→ message::pack()
  │     → 编码 header (12 bytes)
  │     → 编码 questions (域名压缩)
  │     → 编码 answers/authority/additional
  │
  → 发送 UDP 报文

upstream::query_tcp(server, query)
  │
  ├─→ pack_tcp(query)
  │     → [2 bytes length] + message::pack()
  │
  → 发送 TCP 帧

接收响应后:
  │
  ├─→ message::unpack(data)
  │     → 解析 header
  │     → 解析 questions (处理压缩指针)
  │     → 解析 records
  │     → 检测循环引用
  │
  └→ message::extract_ips()
        → 遍历 answers
        → extract_ipv4 / extract_ipv6
```

## 参见

- [[core/resolve/dns/upstream|upstream]] — DNS 查询客户端
- [[core/resolve/dns/detail/cache|cache]] — DNS 结果缓存（使用 qtype）