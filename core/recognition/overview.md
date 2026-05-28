---
tags: [recognition, overview]
layer: core
module: recognition
source:
  - I:/code/Prism/include/prism/recognition/
title: Recognition 模块
updated: 2026-05-27
---

# Recognition 模块

协议识别与伪装方案检测模块，是 Prism 入站流量分发的核心。采用 Probe→Identify→Execute 三阶段流水线。

## 设计决策

### 为什么是 Probe→Identify→Execute 三阶段？

三阶段将"是什么协议"（Probe）、"是哪种伪装方案"（Identify）、"怎么处理"（Execute）解耦。Probe 只需预读 24 字节，成本极低。Identify 仅在 Probe 判定为 TLS 时执行，避免对非 TLS 流量做无意义的 ClientHello 解析。Execute 仅在识别出候选方案后执行握手，成本最高。分层设计确保大多数连接在最低成本阶段完成分流。

**后果**: 非 TLS 流量（HTTP/SOCKS5/SS2022）在 Probe 阶段即完成识别，延迟最低。

### 为什么使用分层检测管道（L1/L2/L3）？

识别伪装方案的成本从低到高：SNI 哈希查找（L1，O(1)）→ session_id 标记检测（L2，内存比较）→ HMAC 验证/熵分析（L3，计算密集）。L3 仅在 L1/L2 产生候选后才执行，避免对每个连接做昂贵计算。

**后果**: 不在 SNI 路由表中的伪装域名只能通过 L2/L3 弱特征识别，识别率下降。

### 识别弱点

| 弱点 | 说明 |
|------|------|
| SS2022 salt 误判 | salt 前两字节为 `0x16 0x03` 时误判为 TLS，概率 ~1/65536 |
| VLESS 弱特征 | 仅 4 字节检测特征，区分度最低 |
| 排除法归类 | 不匹配 SOCKS5/TLS/HTTP 的流量统一归为 Shadowsocks |

## 子模块

| 子模块 | 说明 |
|--------|------|
| [[core/recognition/recognition\|recognition]] | 聚合头文件，统一入口 `recognize()` |
| [[core/recognition/result\|result]] | 分析结果结构 `analysis_result` |
| [[core/recognition/confidence\|confidence]] | 检测置信度枚举 |
| [[core/recognition/layered_pipeline\|layered_pipeline]] | 分层检测管道 L1/L2/L3 |
| [[core/recognition/scheme-route-table\|scheme-route-table]] | SNI 路由表 |
| [[core/recognition/probe/probe\|probe]] | 外层协议探测（24 字节预读） |
| [[core/recognition/probe/analyzer\|analyzer]] | 外层协议检测（纯内存比较） |

## 三阶段流程

| 阶段 | 输入 | 操作 | 输出 |
|------|------|------|------|
| Probe | transmission | 预读 24 字节，首字节特征匹配 | `protocol_type`（HTTP/SOCKS5/TLS/SS2022） |
| Identify | probe_result (仅 TLS) | ClientHello 解析 → 分层检测评分 | `analysis_result`（候选方案 + 置信度） |
| Execute | analysis_result | 按置信度排序执行方案握手 | `shared_transmission`（已建立连接） |

### 阶段间数据传递

| 数据 | 来源 | 去向 | 用途 |
|------|------|------|------|
| `pre_read_data` | Probe | Identify, Execute | ClientHello 解析 / 握手注入 |
| `protocol_type` | Probe | Identify | 决定是否进入 TLS 识别 |
| `analysis_result` | Identify | Execute | 方案选择和执行顺序 |

### 错误处理

| 失败场景 | 处理 |
|----------|------|
| Probe 读取超时/连接关闭 | 返回 unknown → Native 兜底 |
| Identify 无候选 | 执行 Native TLS 握手 |
| Execute 方案握手失败 | 尝试下一候选；全部失败 → Native |

## 调用链

| 上游 | 调用 |
|------|------|
| [[core/instance/session/session\|session]] | `recognition::recognize()` |
| Probe 内部 | `probe::probe()` → `probe::detect()` |
| Identify 内部 | `layered_detection_pipeline` → `stealth::scheme::detect()` |

## 相关模块

- [[core/stealth/overview|Stealth 模块]] — 伪装方案实现
- [[core/channel/transport/transmission|Transport]] — 传输层抽象
