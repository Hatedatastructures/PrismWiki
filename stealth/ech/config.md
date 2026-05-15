     1|---
     2|title: "config — ECH 配置"
     3|source: "include/prism/stealth/ech/config.hpp"
     4|module: "stealth"
     5|submodule: "ech"
     6|type: api
     7|tags: [stealth, ech, config, 配置]
     8|created: 2026-05-15
     9|updated: 2026-05-15
    10|---
    11|
    12|# config.hpp
    13|
    14|> 源码: `include/prism/stealth/ech/config.hpp`
    15|> 模块: [[stealth|stealth]] > [[stealth/ech|ech]]
    16|
    17|## 概述
    18|
    19|ECH（Encrypted Client Hello）配置。ECH 是 TLS 扩展，加密 ClientHello 中的 SNI，防止 SNI 泄露。可以叠加在任意 TLS 伪装协议上（如 Reality、AnyTLS、TrustTunnel）。
    20|
    21|**协议参考**: https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni
    22|
    23|## 依赖关系
    24|
    25|| 依赖方向 | 模块 | 说明 |
    26||----------|------|------|
    27|| 依赖 | [[memory/container|container]] | 使用 memory::string PMR 容器 |
    28|| 被依赖 | [[stealth/anytls/scheme|anytls/scheme]] | 使用 ECH 配置 |
    29|| 被依赖 | [[stealth/executor|executor]] | 执行器使用配置 |
    30|
    31|## 命名空间
    32|
    33|`psm::stealth::ech`
    34|
    35|---
    36|
    37|## 结构体: config
    38|
    39|> 源码: `include/prism/stealth/ech/config.hpp:22`
    40|
    41|### 概述
    42|
    43|ECH 服务端配置。ECH 是叠加层，可叠加在各 TLS 伪装协议上。服务端需要 ECH 密钥配置才能解密 ECH payload。
    44|
    45|### 成员变量
    46|
    47|| 变量 | 类型 | 说明 |
    48||------|------|------|
    49|| ech_key | memory::string | ECH 密钥（base64 编码） |
    50|| public_name | memory::string | 公开的伪装域名 |
    51|
    52|### 设计意图
    53|
    54|- **ech_key**: ECH 密钥，用于解密 ECH payload
    55|- **public_name**: 公开的伪装域名，用于 SNI 路由
    56|
    57|### 生命周期
    58|
    59|1. **构造**: 由配置加载器从 `configuration.json` 解析
    60|2. **使用**: 被 ECH 解密函数和执行器读取
    61|3. **销毁**: 程序退出时自动销毁
    62|
    63|### 线程安全
    64|
    65|- 配置在启动时加载，运行时只读，线程安全
    66|
    67|### 异常安全
    68|
    69|- 不抛异常，配置解析失败时返回默认值
    70|
    71|---
    72|
    73|## 函数: enabled()
    74|
    75|> 源码: `include/prism/stealth/ech/config.hpp:31`
    76|
    77|### 功能
    78|
    79|检查是否启用。`ech_key` 非空时返回 `true`。
    80|
    81|### 签名
    82|
    83|```cpp
    84|[[nodiscard]] auto enabled() const noexcept -> bool;
    85|```
    86|
    87|### 返回值
    88|
    89|`bool` — `ech_key` 非空时返回 `true`，否则返回 `false`
    90|
    91|### 调用链
    92|
    93|**被调用**（向上）:
    94|- [[stealth/ech/decrypt|decrypt]] — 解密前检查是否启用
    95|- [[stealth/executor|executor]] — 执行器检查是否启用
    96|
    97|---
    98|
    99|## JSON 序列化
   100|
   101|配置文件使用 glaze 库进行 JSON 序列化/反序列化。
   102|
   103|### config 结构体字段映射
   104|
   105|| JSON 字段 | C++ 字段 | 类型 |
   106||-----------|----------|------|
   107|| ech_key | ech_key | string |
   108|| public_name | public_name | string |
   109|
   110|### 配置示例
   111|
   112|```json
   113|{
   114|  "stealth": {
   115|    "ech": {
   116|      "ech_key": "base64_encoded_ech_key",
   117|      "public_name": "example.com"
   118|    }
   119|  }
   120|}
   121|```
   122|
   123|---
   124|
   125|## 知识域
   126|
   127|- [[ref/protocol/tls-extensions|TLS 扩展]] — ECH 是 TLS 扩展的一种
   128|
   129|## 相关链接
   130|
   131|- [[stealth/ech/decrypt|decrypt]] — ECH 解密函数，使用此配置
   132|- [[stealth/executor|executor]] — 执行器，使用此配置
   133|- [[ref/protocol/tls-extensions|TLS 扩展]] — ECH 扩展
   134|