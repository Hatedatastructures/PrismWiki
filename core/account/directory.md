---
layer: core
source: include/prism/account/directory.hpp
title: Account Directory
module: account
tags:
  - directory
  - COW
  - transparent-hash
  - lock-free-read
  - CAS
created: 2026-05-23
updated: 2026-05-23
---

# Account Directory

> 源码位置: `include/prism/account/directory.hpp`

## 概述

`directory` 管理凭证 (credential) 到账户条目的映射关系，采用原子共享指针实现**无锁读取**和**写时复制 (Copy-on-Write) 更新**。是 [[core/account/overview|Account 模块]] 的核心存储组件。

## 核心类

### directory

```cpp
namespace psm::account
{
    class directory
    {
    public:
        explicit directory(memory::resource_pointer resource = memory::system::thread_local_pool());

        void reserve(std::size_t n);
        void clear();
        void upsert(std::string_view credential, std::uint32_t max_connections = 0);
        void insert(std::string_view credential, std::shared_ptr<entry> existing_entry);

        [[nodiscard]] auto find(std::string_view credential) const noexcept
            -> std::shared_ptr<entry>;

    private:
        using unordered_map = memory::unordered_map<
            memory::string,
            std::shared_ptr<entry>,
            transparent_hash,
            transparent_equal
        >;

        template <typename UpdateFn>
        void update_entries(UpdateFn &&update_fn);

        memory::allocator<std::byte> allocator_;
        std::atomic<std::shared_ptr<unordered_map>> entries_ptr_;
    };
}
```

## 构造函数

```cpp
explicit directory(memory::resource_pointer resource = memory::system::thread_local_pool());
```

创建空的账户目录。默认使用线程局部内存池，避免多线程竞争。账户目录通常在单线程（main 线程或 loader 线程）中构建，构建完成后通过 `shared_ptr` 传递给 `server` 上下文供所有 worker 线程只读访问。

## 公共方法

### reserve

```cpp
void reserve(std::size_t n);
```

预分配 `n` 个条目的哈希表容量。会触发写时复制（复制当前映射表），适用于批量插入前预分配以减少 rehash 次数。

### clear

```cpp
void clear();
```

原子替换为空映射表。原有条目在所有读取者释放引用后自动销毁。

### upsert

```cpp
void upsert(std::string_view credential, std::uint32_t max_connections = 0);
```

插入或更新账户条目。若凭证不存在则创建新 `entry`，存在则更新 `max_connections`。操作通过 COW 完成：复制当前映射表 → 修改副本 → CAS 原子替换。

### insert

```cpp
void insert(std::string_view credential, std::shared_ptr<entry> existing_entry);
```

将已有 `entry` 以新凭证键注册到目录中。用于实现**多协议凭证共享同一个 entry**，从而共享连接数配额。例如 Trojan UUID 和 SOCKS5 password 可以指向同一个 `entry`。

### find

```cpp
[[nodiscard]] auto find(std::string_view credential) const noexcept -> std::shared_ptr<entry>;
```

无锁读取，通过 `atomic load` 获取当前映射表快照后查询。返回的 `shared_ptr` 可安全跨线程使用。

## 透明哈希机制

directory 使用自定义的 `transparent_hash` 和 `transparent_equal`，支持以 `std::string_view` 为键查询 `memory::string` 为键的映射表，避免临时字符串构造：

### transparent_hash

```cpp
struct transparent_hash
{
    using is_transparent = void;  // 启用透明查找

    [[nodiscard]] auto operator()(const std::string_view value) const noexcept -> std::size_t;
    [[nodiscard]] auto operator()(const memory::string &value) const noexcept -> std::size_t;
};
```

声明 `is_transparent = void` 后，`std::unordered_map` 的 `find()` 模板重载被启用，允许使用异构键类型查找。

### transparent_equal

```cpp
struct transparent_equal
{
    using is_transparent = void;

    [[nodiscard]] auto operator()(const memory::string &left, std::string_view right) const noexcept -> bool;
    [[nodiscard]] auto operator()(const memory::string &left, const memory::string &right) const noexcept -> bool;
    [[nodiscard]] auto operator()(std::string_view left, const memory::string &right) const noexcept -> bool;
};
```

支持 `memory::string` 和 `string_view` 的混合比较，所有比较通过 `string_view` 视图进行，零临时对象。

## 写时复制机制

`update_entries()` 是所有修改操作的底层实现：

```cpp
template <typename UpdateFn>
void update_entries(UpdateFn &&update_fn)
{
    auto current = entries_ptr_.load(std::memory_order_acquire);
    if (!current) {
        current = std::allocate_shared<unordered_map>(allocator_, 0);
    }

    while (true) {
        auto next = std::allocate_shared<unordered_map>(allocator_, *current);
        update_fn(*next);
        if (entries_ptr_.compare_exchange_strong(
                current, next,
                std::memory_order_release,
                std::memory_order_acquire)) {
            return;
        }
    }
}
```

### COW 流程

```
update_entries(upsert "new-user")
    │
    ▼
1. load() 获取当前映射表快照 (shared_ptr)
    │
    ▼
2. allocate_shared 复制整个映射表
    │   ┌────────────────────┐       ┌────────────────────┐
    │   │ current (旧表)     │       │ next (副本)        │
    │   │ "user-a" → entry_a │  ───► │ "user-a" → entry_a │
    │   │ "user-b" → entry_b │       │ "user-b" → entry_b │
    │   └────────────────────┘       └────────┬───────────┘
    │                                         │
    ▼                                         ▼
3. 在副本上执行修改                         │ + "new-user" → entry_c
    │
    ▼
4. CAS 原子替换 entries_ptr_ → next
    │
    ├── 成功: 旧表引用计数递减，无读者后销毁
    └── 失败: 其他线程已修改，重新加载并重试
```

### 并发安全性

- **读取 (find)**: 通过 `atomic load` 获取当前映射表 `shared_ptr`，完全无锁
- **写入 (upsert/insert/clear)**: 通过 CAS 自旋保证原子性
- **ABA 问题**: 每次修改都创建新的 `shared_ptr`，不存在 ABA
- **读者安全**: 读取者持有旧版映射表的 `shared_ptr`，写入者修改不影响正在进行的读取

## 自由函数

### try_acquire

```cpp
[[nodiscard]] inline auto try_acquire(
    const directory &accounts,
    const std::string_view credential) noexcept -> lease;
```

尝试获取账户连接租约。这是连接认证的核心入口点：

1. 调用 `directory::find()` 查找账户
2. 未找到 → 返回空租约
3. `max_connections == 0` → 无限制，直接递增并返回
4. CAS 循环检查 `active_connections < max_connections`
5. 成功 → 递增并返回 `lease`
6. 已满 → 返回空租约

```
try_acquire(dir, "user-credential")
    │
    ├── find("user-credential") → nullptr
    │       └── 返回空 lease → 拒绝连接
    │
    ├── find() → entry{max=0}
    │       └── fetch_add(active_connections, 1)
    │       └── 返回 lease(entry_ptr) → 允许连接
    │
    └── find() → entry{max=10, active=10}
            └── CAS(current, current+1) 失败
            └── current >= max_connections
            └── 返回空 lease → 拒绝连接（已满）
```

### contains

```cpp
[[nodiscard]] inline auto contains(
    const directory &accounts,
    const std::string_view credential) noexcept -> bool;
```

检查账户是否存在，等价于 `find() != nullptr`。

## 与配置加载的集成

```
main.cpp 启动
    │
    ▼
loader::build_account_directory(config)
    │
    ├── 创建 directory 实例
    │
    ├── 遍历 config.authentication.users
    │       │
    │       ├── 首个凭证: dir.upsert(uuid, max_connections)
    │       │       → 创建新 entry
    │       │
    │       └── 后续凭证: dir.insert(password, same_entry)
    │               → 共享同一 entry，共享连接配额
    │
    └── 返回 directory (shared_ptr)
            │
            ▼
        server.account_store = directory
```

## 性能特性

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| `find()` | O(1) 平均 | 哈希表查找 + atomic load |
| `upsert()` | O(n) | 复制整个映射表 + CAS |
| `insert()` | O(n) | 复制整个映射表 + CAS |
| `clear()` | O(1) | 原子替换为空表 |
| `try_acquire()` | O(1) 平均 | find + CAS |

> 写操作 O(n) 复杂度在账户数量少（通常 < 100）的场景下完全可接受。写入仅在配置热加载时发生，运行期间几乎无写入。

## 相关文档

- [[core/account/overview|Account 模块总览]] - 模块架构与设计
- [[core/account/entry|Entry 与 Lease]] - 账户条目与租约定义
- [[core/context/context|Context]] - session 上下文中的 account_directory 指针
- [[core/memory/overview|Memory]] - PMR 容器与内存池
- [[core/loader/load|Loader]] - 配置加载与账户目录构建
- [[core/memory/container|Memory Container]] - PMR unordered_map 定义
