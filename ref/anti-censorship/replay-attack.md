---
title: "重放攻击防御"
category: "anti-censorship"
type: ref
module: ref
source: "概念文档"
tags: [流量对抗, 重放攻击, 安全, 认证, 防护, 防重放]
created: 2026-05-15
updated: 2026-05-17
---

# 重放攻击防御

**类别**: 流量对抗

## 概述

重放攻击（Replay Attack）是一种网络攻击方式，攻击者截获合法的数据包并在稍后时间重新发送，试图使服务器误认为这是新的合法请求。在代理服务场景中，重放攻击可能被用于冒充合法用户、绕过身份验证、或被审查机构用于主动探测识别代理服务器。

重放攻击的本质是利用消息的时效性漏洞。如果一个认证请求在不同时间被多次接受，那么攻击者只需要捕获一次合法请求，就可以反复使用来获得服务。这种攻击不需要破解加密或破解认证机制本身，只需要"复制粘贴"截获的数据。

在审查对抗领域，重放攻击具有特殊的意义。审查机构可能截获真实客户端与代理服务器之间的通信，然后重放这些通信来探测服务器是否运行代理服务。如果服务器对重放的数据包做出与原始请求相同的响应，审查机构就能确认这是一个代理服务器。这使得重放攻击成为主动探测的重要组成部分。

重放攻击的威胁包括：身份冒用（攻击者冒充合法用户）、服务滥用（重复消耗服务器资源）、信息泄露（通过重复请求获取敏感信息）、探测识别（审查机构确认代理服务器）。在代理服务场景中，最严重的是探测识别威胁，这可能导致服务器被封锁。

现代代理协议都实现了某种形式的防重放机制。Shadowsocks 2022 使用时间戳和滑动窗口验证，确保每个数据包只能被接受一次。V2Ray VMess 使用时间戳和命令编号验证。Trojan 使用密码和随机数验证。Reality 使用 short_id 和认证数据验证。

Prism 在 Shadowsocks 2022 和 Reality 协议中实现了完整的防重放机制。通过时间戳验证、滑动窗口、序列号检查等技术，Prism 能够有效抵御重放攻击，同时阻止审查机构的重放探测。

理解重放攻击的原理和防御机制对于设计安全的代理系统至关重要。本文将详细介绍重放攻击的技术原理、攻击类型、防御策略，以及在 Prism 中的实现。

## 原理详解

### 攻击原理

重放攻击的基本原理是利用消息的可复制性。在加密通信中，即使消息内容被加密，攻击者仍然可以复制整个加密数据包并在稍后重新发送。如果服务器不能区分新消息和旧消息，就会接受重放的消息。

```
重放攻击流程:

客户端                              攻击者                    服务器
    |                                   |                       |
    |  1. 发送认证请求                   |                       |
    |----------------------------------->|                       |
    |                                   |                       |
    |                                   |  2. 截获并保存请求     |
    |                                   |                       |
    |                                   |  3. 重放请求          |
    |                                   |---------------------->|
    |                                   |                       |
    |                                   |  4. 服务器接受请求     |
    |                                   |<----------------------|
    |                                   |                       |
    |                                   |  5. 获得服务          |
    |                                   |                       |

问题:
服务器无法区分第 1 次请求和第 3 次请求
两者使用相同的加密数据，内容完全一致
```

**重放攻击的类型**

根据攻击者的动机和能力，重放攻击可以分为多种类型：

**简单重放**

攻击者直接截获并重放原始数据包，不做任何修改。

```
简单重放特征:
- 数据包内容与原始完全一致
- 时间戳是原始时间戳
- 序列号是原始序列号

检测方法:
- 时间戳验证（时间窗口）
- 序列号验证（唯一性）
- 状态记录（已见数据包）
```

**延迟重放**

攻击者在一定时间后重放数据包，可能绕过短期验证。

```
延迟重放特征:
- 数据包内容不变
- 重放时间延迟
- 可能绕过短期窗口

防御方法:
- 扩大时间窗口
- 使用滑动窗口
- 长期状态记录
```

**选择性重放**

攻击者只重放某些关键数据包，如认证请求。

```
选择性重放目标:
- 认证握手包
- 密钥交换包
- 会话建立包

防御方法:
- 不同阶段使用不同验证机制
- 会话绑定验证
- 状态一致性检查
```

**跨会话重放**

攻击者在不同会话之间重放数据包。

```
跨会话重放特征:
- 数据包来自不同会话
- 使用相同认证信息
- 绕过会话隔离

防御方法:
- 会话 ID 绑定
- 会话随机数验证
- 会话状态关联
```

### 审查场景中的重放探测

审查机构使用重放攻击作为主动探测的手段：

```
审查重放探测流程:

正常客户端                          审查探测系统                 代理服务器
    |                                   |                           |
    |  1. 正常代理连接                   |                           |
    |----------------------------------->|                           |
    |                                   |                           |
    |                                   |  2. 截获连接数据           |
    |                                   |                           |
    |                                   |  3. 保存完整握手           |
    |                                   |                           |
    |                                   |  4. 建立探测连接           |
    |                                   |-------------------------->|
    |                                   |                           |
    |                                   |  5. 重放握手数据           |
    |                                   |-------------------------->|
    |                                   |                           |
    |                                   |  6. 分析服务器响应         |
    |                                   |<--------------------------|
    |                                   |                           |
    |                                   |  如果响应与原始相同        |
    |                                   |  → 确认代理服务器          |
```

审查重放探测的目标：
- 确认服务器是否运行代理服务
- 绕过加密保护识别协议类型
- 验证 DPI 的检测结果
- 收集代理协议特征

审查重放探测的特点：
- 使用真实客户端数据（最隐蔽）
- 可以绕过基于 IP 的过滤
- 可以绕过简单的身份验证
- 需要较强的截获能力

### 防御机制

现代代理协议使用多种机制防御重放攻击：

**时间戳验证**

时间戳验证是最基本的防重放机制。每个数据包包含发送时的时间戳，服务器验证时间戳是否在有效窗口内。

```
时间戳验证流程:

客户端                                    服务器
    |                                       |
    |  数据包 + 时间戳 (T_client)           |
    |-------------------------------------->|
    |                                       |
    |                                       |  1. 提取时间戳
    |                                       |  2. 获取当前时间 T_server
    |                                       |  3. 计算 |T_server - T_client|
    |                                       |  4. 验证是否 <= 窗口大小
    |                                       |
    |                                       |  如果有效 → 接受
    |                                       |  如果无效 → 拒绝

时间窗口设置:
- 短窗口: ±30 秒（高安全）
- 中窗口: ±60 秒（平衡）
- 长窗口: ±120 秒（宽松）

窗口过短的问题:
- 时钟偏移导致拒绝
- 网络延迟导致拒绝
- 需要精确的时间同步

窗口过长的问题:
- 增加重放攻击窗口
- 降低安全性
- 可能被探测者利用
```

**滑动窗口**

滑动窗口是一种更精确的防重放机制。服务器记录已接受的数据包时间戳，只接受窗口内未见过的数据包。

```
滑动窗口机制:

                    滑动窗口
                    +-------+
    过去  ←---------|  当前 |---------→ 未来
                    +-------+

窗口内已见时间戳:
| T-60 | T-55 | T-50 | T-45 | ... | T+30 | T+35 | T+40 |
+------+------+------+------+-----+------+------+------+
|  ✓   |  ✓   |  ✓   |      | ... |      |      |      |

规则:
- 窗口内的已见时间戳 → 拒绝（重放）
- 窗口内的未见时间戳 → 接受（新包）
- 窗口外的时间戳 → 拒绝（过期）

窗口滑动:
- 随时间向前移动
- 最旧的记录被淘汰
- 新的记录被添加

实现:
- 使用环形缓冲区
- 使用位图标记已见
- 使用哈希表存储
```

**序列号验证**

序列号验证为每个数据包分配唯一的递增序列号，确保数据包的唯一性和顺序性。

```
序列号验证机制:

客户端发送序列:
Seq: 1, 2, 3, 4, 5, 6, ...

服务器接受:
- Seq 1: 接受，记录
- Seq 2: 接受，记录
- Seq 3: 接受，记录
- Seq 2 (重放): 拒绝，已见
- Seq 4: 接受，记录
- Seq 7 (跳跃): 检查策略

处理策略:
- 精确递增: 只接受 Seq = last_seq + 1
- 窗口递增: 接受窗口内的未见序列号
- 大跳跃: 可能丢包，需要处理

序列号存储:
- 位图: 快速查找，固定窗口
- 哈希表: 无限窗口，内存消耗
- 环形缓冲: 平衡方案
```

**一次性令牌**

一次性令牌（Nonce）确保每个请求只能使用一次。令牌由服务器或客户端生成，使用后立即失效。

```
一次性令牌机制:

服务器生成令牌:
客户端                              服务器
    |                                   |
    |  请求连接                          |
    |---------------------------------->|
    |                                   |
    |                                   |  生成令牌 N
    |  令牌 N                            |
    |<----------------------------------|
    |                                   |
    |  认证请求 + 令牌 N                 |
    |---------------------------------->|
    |                                   |
    |                                   |  验证令牌
    |                                   |  标记已用
    |  认证成功                          |
    |<----------------------------------|
    |                                   |
    |                                   |  令牌 N 已失效

客户端生成令牌:
客户端                              服务器
    |                                   |
    |  认证请求 + 客户端令牌 C           |
    |---------------------------------->|
    |                                   |
    |                                   |  检查令牌是否已用
    |                                   |  记录令牌
    |                                   |
    |  认证结果                          |
    |<----------------------------------|

令牌类型:
- 随机数: 完全随机，碰撞概率低
- 序列号: 递增序列，简单管理
- 时间戳+随机数: 混合方案，平衡性能
```

**会话绑定**

会话绑定将数据包与特定会话关联，防止跨会话重放。

```
会话绑定机制:

会话建立阶段:
客户端                              服务器
    |                                   |
    |  会话建立请求                      |
    |---------------------------------->|
    |                                   |
    |                                   |  生成会话 ID
    |  会话 ID (SID)                     |
    |<----------------------------------|
    |                                   |

数据传输阶段:
客户端                              服务器
    |                                   |
    |  数据包 + SID                      |
    |---------------------------------->|
    |                                   |
    |                                   |  验证 SID
    |                                   |  验证数据包在 SID 内唯一
    |                                   |

跨会话重放防护:
- 数据包必须包含正确的 SID
- SID 与连接绑定
- 不同连接无法重放旧数据包

SID 生成:
- 随机数: 每次连接生成新的 SID
- 连接特征: 使用 IP+端口+时间戳组合
- 客户端标识: 使用客户端证书或密钥
```

### 协议级别的防重放

各主流代理协议的防重放机制：

**Shadowsocks 2022**

Shadowsocks 2022 使用时间戳和滑动窗口实现防重放：

```
Shadowsocks 2022 数据包结构:
+------------------+------------------+------------------+------------------+
| 时间戳 (8 字节)  | PSK 标记 (8 字节)| 类型 (1 字节)    | 载荷 (变长)      |
+------------------+------------------+------------------+------------------+

时间戳处理:
- 使用 Unix 时间戳（秒）
- 精确到秒级
- 窗口: ±60 秒

滑动窗口:
- 环形缓冲区记录已见时间戳
- 窗口大小: 120 秒
- 位图标记已见状态
```

**V2Ray VMess**

VMess 使用时间戳和命令编号实现防重放：

```
VMess 请求结构:
+------------------+------------------+------------------+------------------+
| 版本 (1 字节)    | 时间戳 (8 字节)  | 命令编号 (4 字节)| 载荷 (加密)      |
+------------------+------------------+------------------+------------------+

时间戳处理:
- 使用 Unix 时间戳（秒）
- 窗口: ±120 秒

命令编号:
- 每个请求唯一
- 递增序列
- 防止重复请求

认证:
- 用户 ID + 时间戳 + 命令编号
- HMAC 验证
```

**Trojan**

Trojan 使用密码和随机数实现防重放：

```
Trojan 请求结构:
+------------------+------------------+------------------+------------------+
| SHA224(密码)     | 随机数 (16 字节) | 命令 (1 字节)    | 载荷             |
+------------------+------------------+------------------+------------------+

密码验证:
- SHA224 哈希密码
- 固定长度 28 字节

随机数:
- 每次请求生成新随机数
- 16 字节长度
- 防止重放

注意:
- 原版 Trojan 随机数不验证
- 存在重放风险
- 需要额外实现滑动窗口
```

## 在 Prism 中的应用

Prism 在多个协议中实现了完整的防重放机制：

### Shadowsocks 2022 防重放

Prism 实现了 Shadowsocks 2022 的滑动窗口防重放机制：

```cpp
// include/prism/protocol/shadowsocks/replay.hpp

/// @brief 防重放滑动窗口
/// @details 使用位图记录已见时间戳，防止重放攻击
class replay_window {
public:
    /// @brief 构造函数
    /// @param window_size 窗口大小（秒）
    explicit replay_window(size_t window_size = 120);

    /// @brief 检查并记录时间戳
    /// @param timestamp Unix 时间戳
    /// @return true 如果时间戳有效且未见，false 如果时间戳无效或已见
    auto check_and_record(uint64_t timestamp) -> bool;

    /// @brief 清理过期记录
    void cleanup();

private:
    size_t window_size_;                  ///< 窗口大小
    uint64_t window_base_;                ///< 窗口起始时间
    std::vector<bool> seen_map_;          ///< 已见时间戳位图
    std::mutex mutex_;                    ///< 保护位图

    auto is_in_window(uint64_t timestamp) const -> bool;
    auto get_window_offset(uint64_t timestamp) const -> size_t;
};
```

实现细节：

```cpp
// src/prism/protocol/shadowsocks/replay.cpp

auto replay_window::check_and_record(uint64_t timestamp) -> bool {
    auto current_time = get_current_timestamp();

    // 1. 检查时间戳是否在有效窗口内
    if (timestamp < current_time - window_size_ / 2 ||
        timestamp > current_time + window_size_ / 2) {
        return false;  // 时间戳过期或超前太多
    }

    // 2. 滑动窗口
    if (timestamp > window_base_ + window_size_) {
        // 需要滑动窗口
        auto new_base = timestamp - window_size_ / 2;
        auto slide_distance = new_base - window_base_;

        // 清理旧记录
        for (size_t i = 0; i < slide_distance && i < seen_map_.size(); ++i) {
            seen_map_[i] = false;
        }

        window_base_ = new_base;
    }

    // 3. 检查是否已见
    auto offset = get_window_offset(timestamp);
    if (offset >= seen_map_.size()) {
        return false;
    }

    if (seen_map_[offset]) {
        return false;  // 已见，重放攻击
    }

    // 4. 记录已见
    seen_map_[offset] = true;
    return true;
}
```

### Reality 防重放

Reality 协议使用 short_id 和认证数据实现防重放：

```cpp
// src/prism/stealth/reality/auth.cpp

auto reality_authenticator::verify_auth_data(
    const auth_data& data,
    const reality_config& config
) -> bool {
    // 1. 验证 short_id
    if (!config.short_ids.contains(data.short_id)) {
        return false;
    }

    // 2. 验证时间戳
    auto current_time = get_current_timestamp();
    if (data.timestamp < current_time - timestamp_window_ ||
        data.timestamp > current_time + timestamp_window_) {
        return false;  // 时间戳无效
    }

    // 3. 检查重放窗口
    if (!replay_window_.check_and_record(data.timestamp)) {
        return false;  // 重放攻击
    }

    // 4. 验证认证哈希
    auto expected_hash = compute_auth_hash(
        data.short_id,
        data.timestamp,
        config.public_key
    );

    return data.auth_hash == expected_hash;
}
```

### ShadowTLS 防重放

ShadowTLS v3 使用密码哈希实现防重放：

```cpp
// src/prism/stealth/shadowtls/auth.cpp

auto shadowtls_authenticator::verify_password_hash(
    const std::vector<uint8_t>& received_hash,
    const shadowtls_config& config
) -> bool {
    auto current_time = get_current_timestamp();

    // 1. 检查当前时间窗口
    for (int offset = -time_window_; offset <= time_window_; ++offset) {
        auto expected_hash = compute_password_hash(
            config.password,
            current_time + offset
        );

        if (received_hash == expected_hash) {
            // 2. 检查是否已使用该时间戳
            if (!replay_window_.check_and_record(current_time + offset)) {
                return false;  // 重放攻击
            }
            return true;
        }
    }

    return false;  // 密码不匹配
}

auto shadowtls_authenticator::compute_password_hash(
    const std::string& password,
    uint64_t timestamp
) -> std::vector<uint8_t> {
    // 使用 HMAC-SHA256 计算密码哈希
    auto key = derive_key_from_password(password);
    auto message = fmt::format("{}:{}", password, timestamp);

    return crypto::hmac_sha256(key, message);
}
```

### 配置示例

Shadowsocks 2022 配置：

```json
{
  "protocol": {
    "shadowsocks": {
      "psk": "BASE64_ENCODED_PSK",
      "method": "2022-blake3-aes-128-gcm",
      "replay_protection": {
        "enabled": true,
        "window_size": 120,
        "cleanup_interval": 60
      }
    }
  }
}
```

Reality 配置：

```json
{
  "stealth": {
    "reality": {
      "short_ids": [
        "0123456789abcdef",
        "fedcba9876543210"
      ],
      "replay_protection": {
        "enabled": true,
        "timestamp_window": 60
      }
    }
  }
}
```

### 防重放状态管理

Prism 使用高效的内存管理来存储防重放状态：

```cpp
// include/prism/protocol/shadowsocks/replay_window.hpp

/// @brief 防重放窗口配置
struct replay_config {
    bool enabled = true;           ///< 是否启用防重放
    size_t window_size = 120;      ///< 窗口大小（秒）
    size_t cleanup_interval = 60;  ///< 清理间隔（秒）
    size_t max_memory = 1048576;   ///< 最大内存使用（字节）
};

/// @brief 全局防重放窗口管理器
class replay_manager {
public:
    /// 获取或创建指定密钥的防重放窗口
    auto get_window(const std::string& key) -> std::shared_ptr<replay_window>;

    /// 清理所有窗口
    void cleanup_all();

    /// 获取内存使用统计
    auto get_memory_usage() const -> size_t;

private:
    std::unordered_map<std::string, std::shared_ptr<replay_window>> windows_;
    std::mutex mutex_;
    replay_config config_;
};
```

## 最佳实践

### 配置建议

**时间窗口选择**

根据场景选择合适的时间窗口：

| 场景 | 建议窗口 | 原因 |
|------|----------|------|
| 高安全（金融、企业） | ±30 秒 | 最小化重放窗口 |
| 一般用途 | ±60 秺 | 平衡安全和可用性 |
| 高延迟网络 | ±120 秒 | 允许网络延迟 |
| 宽松部署 | ±300 秒 | 最大兼容性 |

**内存管理**

合理配置内存限制：

```json
{
  "replay_protection": {
    "max_memory": 1048576,  // 1MB
    "window_size": 120,
    "cleanup_interval": 60
  }
}
```

内存估算：
- 每个时间戳: 1 位
- 窗口大小 120 秒: 120 位 = 15 字节
- 1000 个密钥: 15KB
- 安全上限: 1MB

**时钟同步**

确保服务器时钟准确：
- 使用 NTP 同步时间
- 定期检查时钟偏移
- 允许合理的时钟误差

### 实现建议

**高效实现**

使用位图而非哈希表：
- 位图: O(1) 查找，固定内存
- 哈希表: O(1) 查找，无限内存增长
- 环形缓冲: O(1) 查找，固定内存，自动清理

**并发安全**

在多线程环境中保护防重放状态：
- 使用 atomic 操作
- 使用 mutex 保护复杂操作
- 使用 per-thread 窗口减少锁竞争

**性能监控**

监控防重放机制的性能：
- 记录拒绝率
- 分析拒绝原因
- 监控内存使用

## 常见问题

### Q1: 防重放会拒绝正常请求吗？

可能会。在以下情况下正常请求可能被拒绝：
- 时钟偏移过大（服务器和客户端时间不同）
- 网络延迟过高（请求到达时已过期）
- 并发请求序列号跳跃

解决方案：
- 放宽时间窗口
- 使用 NTP 同步时钟
- 实现跳跃序列号处理

### Q2: 如何处理丢包？

丢包会导致序列号跳跃，处理策略：
- 精确递增模式：拒绝跳跃，等待重传
- 窗口递增模式：接受窗口内未见序列号
- 容错模式：记录跳跃，继续接受

推荐使用窗口递增模式。

### Q3: 内存消耗如何控制？

控制方法：
- 固定窗口大小（位图）
- 定期清理过期记录
- 设置最大内存限制
- 使用 LRU 淘汰策略

### Q4: 如何应对时钟攻击？

时钟攻击是攻击者操纵时间戳来绕过防重放：
- 使用服务器时间而非客户端时间
- 检测时间戳异常模式
- 使用加密的时间戳

### Q5: 重放探测能被完全阻止吗？

不能完全阻止，但可以有效限制：
- 审查者可能截获并立即重放
- 短时间窗口内的重放难以阻止
- 可以增加探测者的成本

目标是增加探测成本，而非完全阻止。

### Q6: Reality 如何防止重放探测？

Reality 使用多层机制：
- short_id 验证（已知客户端）
- 时间戳验证（时效性）
- 滑动窗口（唯一性）
- 回退机制（对探测者返回正常响应）

### Q7: Shadowsocks 2022 比 2018 更安全吗？

是的，防重放是主要改进之一：
- 2022: 完整的滑动窗口防重放
- 2018: 无防重放机制，容易被探测

推荐使用 Shadowsocks 2022。

## 参考资料

- [Replay Attack - Wikipedia](https://en.wikipedia.org/wiki/Replay_attack)
- [Shadowsocks 2022 Specification](https://github.com/Shadowsocks-NET/shadowsocks-net)
- [VMess Protocol](https://www.v2fly.org/developer/protocol/vmess.html)
- [Trojan Protocol](https://trojan-gfw.github.io/trojan/protocol)
- [Reality Protocol](https://github.com/XTLS/REALITY)
- [Nonce - Wikipedia](https://en.wikipedia.org/wiki/Cryptographic_nonce)

## 相关知识

- [[ref/anti-censorship/dpi|深度包检测]] — DPI 技术
- [[ref/anti-censorship/probing|主动探测]] — 主动探测防御
- [[ref/anti-censorship/mitm-hijack|中间人劫持]] — MITM 攻击防御
- [[ref/protocol/shadowsocks|Shadowsocks 协议]] — Shadowsocks 实现
- [[prism/stealth/reality|Reality 协议]] — Reality 实现细节
- [[prism/crypto/hmac|HMAC]] — HMAC 实现