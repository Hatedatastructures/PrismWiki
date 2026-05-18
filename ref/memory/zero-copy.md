---
title: "零拷贝"
category: "memory"
type: ref
module: ref
source: "概念文档"
tags: [内存, 零拷贝, 性能, sendfile, mmap, splice, io_uring, 异步IO]
created: 2026-05-15
updated: 2026-05-17
related:
  - ref/memory/pmr
  - ref/memory/arena
  - channel/transport/transmission
layer: ref
---

# 零拷贝

**类别**: 内存

## 概述

### 什么是零拷贝

零拷贝（Zero-copy）是一种数据传输优化技术，旨在消除数据在不同内存区域之间的不必要复制。在传统的数据传输路径中，数据往往需要经过多次复制才能到达目的地：从磁盘到内核缓冲区、从内核缓冲区到用户缓冲区、从用户缓冲区到 Socket 缓冲区、最后从 Socket 缓冲区到网卡。零拷贝技术通过优化这些路径，将复制次数减少到最低限度，甚至完全消除复制。

零拷贝的核心思想是"让数据停留在原地，只传递数据的引用或描述符"。通过内核与用户空间的协作，数据可以直接从源设备传输到目标设备，无需经过应用程序的中间缓冲区。这种设计显著降低了 CPU 开销、内存带宽消耗和上下文切换次数。

### 传统数据传输路径

在传统模式下，从磁盘读取文件并通过网络发送的数据流路径如下：

```
传统数据传输路径（4 次复制 + 4 次上下文切换）:

1. 用户调用 read():
   磁盘 --> DMA --> 内核读缓冲区
   (DMA 复制)

2. 内核复制到用户空间:
   内核读缓冲区 --> CPU 复制 --> 用户缓冲区
   (CPU 复制)

3. 用户调用 write():
   用户缓冲区 --> CPU 复制 --> Socket 缓冲区
   (CPU 复制)

4. 发送到网卡:
   Socket 缓冲区 --> DMA --> 网卡
   (DMA 复制)

总计: 4 次数据复制（2 次 DMA + 2 次 CPU）
      4 次上下文切换（read + write 各 2 次）
```

**具体步骤分析**：

1. **read() 系统调用**：用户进程调用 read()，触发上下文切换从用户态到内核态
2. **DMA 读取**：磁盘控制器通过 DMA 将数据直接写入内核读缓冲区
3. **CPU 复制**：内核将数据从内核缓冲区复制到用户缓冲区
4. **read() 返回**：上下文切换回用户态，数据现在在用户空间
5. **write() 系统调用**：用户进程调用 write()，再次切换到内核态
6. **CPU 复制**：内核将数据从用户缓冲区复制到 Socket 缓冲区
7. **DMA 发送**：网卡控制器通过 DMA 从 Socket 缓冲区发送数据
8. **write() 返回**：上下文切换回用户态

**问题分析**：

- **CPU 参与复制**：步骤 3 和 6 需要 CPU 执行内存复制，消耗 CPU 周节
- **内存带宽浪费**：数据在内存中往返，占用内存带宽
- **上下文切换开销**：每次系统调用都涉及上下文切换
- **缓存污染**：数据复制过程污染 CPU 缓存，影响其他计算

### 零拷贝技术分类

零拷贝技术可分为以下几类：

**硬件层面零拷贝**：
- **DMA（Direct Memory Access）**：设备直接访问内存，无需 CPU 参与
- **RDMA（Remote DMA）**：网络设备直接访问远程主机内存

**操作系统层面零拷贝**：
- **sendfile**：内核直接从文件传输到 Socket
- **mmap**：内存映射，用户空间直接访问内核缓冲区
- **splice**：管道传输，数据在内核内移动
- **tee**：数据复制到多个目标

**编程层面零拷贝**：
- **引用传递**：传递数据引用而非数据本身
- **缓冲区共享**：多模块共享同一缓冲区
- **PMR 容器移动**：C++ 容器移动语义避免复制

### 零拷贝的目标

零拷贝技术追求的目标包括：

**减少 CPU 参与**：让 DMA 或专用硬件处理数据移动，释放 CPU 进行其他计算。

**减少内存复制**：从 4 次复制减少到 2 次、1 次，甚至 0 次。

**减少上下文切换**：减少系统调用次数，降低切换开销。

**减少内存占用**：避免冗余缓冲区，节省内存空间。

**提高吞吐量**：更高的数据传输速率，支持高并发场景。

### 适用场景

零拷贝特别适合以下场景：

**代理服务器**：从上游接收数据转发到下游，数据不经过应用程序处理。

**静态文件服务**：从磁盘读取文件发送给客户端，无需修改文件内容。

**流媒体传输**：视频、音频数据流从存储传输到网络。

**大数据传输**：TB 级数据在服务器间传输，复制开销巨大。

**消息队列**：消息在生产者和消费者间传递，避免复制。

### Prism 中的应用动机

Prism 作为高性能代理服务器，大量数据需要从上游转发到下游：

**隧道转发**：TCP 连接间的双向透明转发，数据无需解析或修改。

**协议处理**：部分协议（如 Trojan、Shadowsocks）只添加/去除头部，数据主体可直接转发。

**内存效率**：高并发场景下，避免数据复制可以显著降低内存带宽压力。

**延迟优化**：减少复制和上下文切换，降低转发延迟。

---

## 原理详解

### sendfile 系统调用

sendfile 是最经典的零拷贝技术，Linux 2.2 引入，后续版本不断优化。

**基本接口**：

```c
#include <sys/sendfile.h>

ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

**参数说明**：
- `out_fd`：目标文件描述符（必须是 Socket）
- `in_fd`：源文件描述符（必须是支持 mmap 的文件）
- `offset`：源文件起始偏移（可为 NULL，表示从当前位置开始）
- `count`：传输字节数

**早期 sendfile 路径（Linux 2.2 - 2.4）**：

```
早期 sendfile（2 次复制）:

磁盘 --> DMA --> 内核读缓冲区 --> CPU --> Socket 缓冲区 --> DMA --> 网卡

仍需 CPU 从内核读缓冲区复制到 Socket 缓冲区
```

**改进后的 sendfile 路径（Linux 2.6+）**：

```
现代 sendfile（真正零拷贝）:

磁盘 --> DMA --> 内核读缓冲区
           |
           | (描述符传递，无复制)
           v
        Socket 缓冲区 --> DMA --> 网卡

步骤:
1. DMA 从磁盘读取到内核读缓冲区
2. 内核将读缓冲区的描述符（位置、长度）写入 Socket 缓冲区
3. DMA 根据 Socket 缓冲区的描述符直接从读缓冲区发送到网卡
```

**关键技术：描述符传递**：

现代 sendfile 不复制数据本身，而是在 Socket 缓冲区中存储指向内核读缓冲区的指针和长度信息。DMA 网卡驱动可以理解这些描述符，直接从读缓冲区读取数据发送。

**性能提升**：

| 指标 | 传统 read/write | sendfile |
|------|----------------|----------|
| CPU 复制次数 | 2 次 | 0 次 |
| DMA 复制次数 | 2 次 | 2 次 |
| 上下文切换次数 | 4 次 | 2 次（仅 sendfile 调用） |
| 用户缓冲区需求 | 需要 | 不需要 |

### mmap 内存映射

mmap 将文件映射到进程的虚拟地址空间，用户可以直接访问文件内容。

**基本接口**：

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
int munmap(void *addr, size_t length);
```

**参数说明**：
- `addr`：建议映射地址（通常为 NULL，由内核决定）
- `length`：映射长度
- `prot`：保护标志（PROT_READ、PROT_WRITE、PROT_EXEC）
- `flags`：映射类型（MAP_SHARED、MAP_PRIVATE）
- `fd`：文件描述符
- `offset`：文件偏移

**mmap 工作原理**：

```
mmap 内存布局:

进程虚拟地址空间:
+------------------------+
| 用户缓冲区 (可访问)     |
+------------------------+
| mmap 区域              | <-- 映射到文件
| (虚拟地址)             |     (访问时触发缺页中断)
+------------------------+

访问 mmap 区域:
1. 用户访问虚拟地址
2. MMU 发现页表项无效（未映射物理页）
3. 触发缺页中断
4. 内核分配物理页，从文件读取数据
5. 更新页表，映射物理页
6. 返回用户继续执行

后续访问: 直接访问物理页，无需再次缺页
```

**mmap 数据传输路径**：

```
mmap + write（1 次 CPU 复制）:

磁盘 --> DMA --> 内核页缓存
                    |
                    | (mmap 共享此区域)
                    v
                用户虚拟地址
                    |
                    | write() CPU 复制
                    v
                Socket 缓冲区 --> DMA --> 网卡
```

**mmap 的优缺点**：

优点：
- 用户直接访问内核缓冲区，避免一次复制
- 支持随机访问文件任意位置
- 多进程可以共享同一映射（MAP_SHARED）

缺点：
- 仍需一次 CPU 复制（write 到 Socket 缓冲区）
- 文件大小变化可能导致映射失效
- 映射开销（页表、缺页中断）可能大于小文件复制的开销

### splice 系统调用

splice 在两个文件描述符之间移动数据，无需经过用户空间。

**基本接口**：

```c
#include <linux/splice.h>

ssize_t splice(int fd_in, loff_t *off_in, int fd_out,
               loff_t *off_out, size_t len, unsigned int flags);
```

**参数说明**：
- `fd_in`：输入文件描述符
- `off_in`：输入偏移
- `fd_out`：输出文件描述符
- `off_out`：输出偏移
- `len`：传输长度
- `flags`：控制标志（SPLICE_F_MOVE、SPLICE_F_MORE、SPLICE_F_GIFT）

**splice 约束**：
- 至少一端必须是管道（pipe）
- 支持管道到管道、管道到 Socket、文件到管道

**splice 工作原理**：

```
splice 传输路径:

源文件 --> splice --> 管道 --> splice --> Socket

内部机制:
1. 内核从源文件读取数据到内核缓冲区
2. 内核将缓冲区指针写入管道（不复制数据）
3. 内核从管道读取指针
4. 内核将指针写入 Socket 缓冲区（不复制数据）
5. DMA 从内核缓冲区发送到网卡

数据始终在内核空间，无需复制到用户空间
```

**典型用法（文件到 Socket）**：

```c
// 创建管道
int pipefd[2];
pipe(pipefd);

// 文件到管道
splice(file_fd, NULL, pipefd[1], NULL, len, 0);

// 管道到 Socket
splice(pipefd[0], NULL, socket_fd, NULL, len, 0);
```

**splice 的优势**：

- 完全在内核空间完成，零用户空间复制
- 管道作为中间层，解耦源和目标
- 支持任意大小的传输

### tee 系统调用

tee 将数据从一个管道复制到另一个管道，同时保留原始数据。

**基本接口**：

```c
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);
```

**tee 用途**：

- 数据需要发送到多个目标时
- 日志记录 + 数据转发
- 监控 + 正常传输

```
tee 数据流:

源 --> 管道A --> tee --> 管道B --> 目标1
          |
          | tee 复制
          v
       管道C --> 目标2
```

### io_uring 零拷贝

io_uring 是 Linux 5.1 引入的高性能异步 I/O 接口，支持更高效的零拷贝。

**核心机制**：

- **共享环形缓冲区**：内核和用户共享提交队列（SQ）和完成队列（CQ）
- **无系统调用**：用户直接写入 SQ，内核轮询 SQ 执行操作
- **零拷贝读取**：支持直接将数据写入用户缓冲区，无需中间缓冲

**io_uring 数据路径**：

```
io_uring 零拷贝读取:

用户缓冲区 <-- DMA <-- 内核页缓存 <-- DMA <-- 猫盘

内核将用户缓冲区信息放入 SQ
DMA 直接从页缓存读取到用户缓冲区
```

**io_uring send**：

```
io_uring 零拷贝发送:

内核页缓存 --> DMA --> 网卡

类似 sendfile，但通过 SQ 提交，无需系统调用
```

### Windows 零拷贝技术

Windows 提供类似的零拷贝机制：

**TransmitFile**：

```c
BOOL TransmitFile(
  SOCKET hSocket,
  HANDLE hFile,
  DWORD nNumberOfBytesToWrite,
  DWORD nNumberOfBytesPerSend,
  LPOVERLAPPED lpOverlapped,
  LPTRANSMIT_FILE_BUFFERS lpTransmitBuffers,
  DWORD dwFlags
);
```

类似 Linux sendfile，从文件传输到 Socket。

**WSARecvMsg + Scatter-Gather**：

Windows Socket API 支持 scatter-gather I/O，一次系统调用传输多个缓冲区。

### RDMA 远程直接内存访问

RDMA（Remote Direct Memory Access）允许网络设备直接访问远程主机内存。

**RDMA 特性**：

- **零 CPU 参与**：网卡 DMA 直接读写远程内存
- **零协议开销**：绕过内核网络栈，用户态直接访问网卡
- **零复制**：数据直接从源内存到目标内存

**RDMA 协议**：

- **InfiniBand**：高性能 RDMA 网络
- **RoCE**：以太网上的 RDMA
- **iWARP**：TCP/IP 上的 RDMA

```
RDMA 数据路径:

主机A 应用缓冲区 <-- RDMA NIC <-- 网络 --> RDMA NIC --> 主机B 应用缓冲区

完全绕过内核，零复制，零上下文切换
```

### 编程层面零拷贝

**C++ 移动语义**：

```cpp
// 避免 vector 复制
std::vector<int> source = {...};
std::vector<int> target = std::move(source);  // 移动而非复制

// 避免 string 复制
std::string process(std::string input) {
  return input;  // 自动移动，无复制
}
```

**PMR 容器共享**：

```cpp
// 多模块共享同一缓冲区
pmr::vector<uint8_t> buffer(pool);

// 模块1 处理前半部分
process_front(buffer);

// 模块2 处理后半部分
process_back(buffer);

// 无数据复制，仅传递引用或指针
```

**引用传递**：

```cpp
// 使用引用而非值传递
void process(const std::vector<uint8_t>& data) {
  // 接收引用，不复制数据
}

// 使用 span 传递视图
void process(std::span<uint8_t> data) {
  // 传递视图，不复制数据
}
```

---

## 在 Prism 中的应用

### 隧道传输优化

Prism 的核心功能是隧道传输，将上游连接的数据透明转发到下游连接：

**传统隧道实现**：

```cpp
// 需要数据复制的方式
net::awaitable<void> tunnel_with_copy(tcp_socket& upstream, tcp_socket& downstream) {
  std::vector<char> buffer(65536);

  while (true) {
    auto n = co_await upstream.async_read_some(net::buffer(buffer));
    co_await net::async_write(downstream, net::buffer(buffer, n));
  }
}
```

**优化后的隧道实现**：

```cpp
// 使用 PMR 容器减少复制
net::awaitable<void> tunnel_optimized(tcp_socket& upstream, tcp_socket& downstream,
                                       frame_arena& arena) {
  // 使用帧分配器，避免堆分配
  memory::vector<char> buffer(arena.get());
  buffer.resize(65536);

  // 协程内复用同一缓冲区
  while (true) {
    auto n = co_await upstream.async_read_some(net::buffer(buffer));
    co_await net::async_write(downstream, net::buffer(buffer.data(), n));
  }
}
```

### 异步 I/O 集成

Prism 使用 Boost.Asio 进行异步 I/O，Asio 本身支持零拷贝特性：

**scatter-gather I/O**：

```cpp
// 多缓冲区一次系统调用
std::array<net::const_buffer, 3> buffers = {
  net::buffer(header),
  net::buffer(body),
  net::buffer(trailer)
};
co_await net::async_write(socket, buffers);  // 单次系统调用
```

**缓冲区复用**：

```cpp
// 连接池复用缓冲区
class connection {
  memory::vector<char> read_buffer_;
  memory::vector<char> write_buffer_;

  void reuse_buffers() {
    // 不释放缓冲区，复用于下一个请求
  }
};
```

### PMR 容器零拷贝

Prism 使用 PMR 容器实现数据共享：

**协议处理器间共享**：

```cpp
// 协议解析结果共享
struct parse_result {
  memory::vector<uint8_t> payload;  // 使用 PMR，可跨模块共享
  memory::string hostname;
};

void process_protocol(parse_result& result) {
  // result 的 payload 直接传递给隧道，无需复制
  tunnel(result.payload);
}
```

**避免临时容器复制**：

```cpp
// 使用移动语义
memory::vector<uint8_t> parse(frame_arena& arena) {
  memory::vector<uint8_t> result(arena.get());
  // 解析逻辑
  return result;  // 移动返回，不复制
}
```

### 内存池与零拷贝

Prism 的内存池设计与零拷贝配合：

**连接级缓冲池**：

```cpp
// 每个连接持有独立缓冲区，避免临时分配
class session {
  memory::vector<char> upstream_buffer_;
  memory::vector<char> downstream_buffer_;

  // 缓冲区生命周期与 session 一致
  // 无每次读写的分配开销
};
```

**帧级批量处理**：

```cpp
// 一批请求共享帧缓冲
net::awaitable<void> process_batch(frame_arena& arena) {
  memory::vector<request> requests(arena.get());

  // 批量读取
  for (auto& req : requests) {
    co_await read_request(req);
  }

  // 批量处理，无单独分配
  // 帧结束时统一释放
}
```

### 传输层实现

Prism 的 transmission 模块是隧道转发的核心：

**设计原则**：

- 使用固定大小缓冲区，避免动态分配
- 异步读写复用同一缓冲区
- 协程挂起时保留缓冲区所有权

**示例实现**：

```cpp
namespace psm::channel::transport {
  class transmission {
    static constexpr size_t buffer_size = 64 * 1024;
    std::array<char, buffer_size> buffer_;  // 固定大小栈缓冲

    net::awaitable<void> forward(tcp_socket& src, tcp_socket& dst) {
      while (true) {
        auto n = co_await src.async_read_some(net::buffer(buffer_));
        if (n == 0) break;
        co_await net::async_write(dst, net::buffer(buffer_, n));
      }
    }
  };
}
```

---

## 性能分析

### 数据复制开销量化

**CPU 复制开销**：

每次内存复制消耗的 CPU 周节（取决于大小）：

| 数据大小 | 复制耗时（CPU 周节） | 说明 |
|---------|---------------------|------|
| 64B | 约 20-30 | 小块，可能完全在缓存内 |
| 1KB | 约 50-100 | 中块，部分缓存命中 |
| 16KB | 约 300-500 | 大块，缓存 miss 较多 |
| 64KB | 约 1000-2000 | 超大块，带宽瓶颈 |
| 1MB | 约 10000-20000 | 流传输级别 |

**零拷贝节省**：

以 64KB 数据传输为例：

| 方式 | CPU 复制次数 | CPU 周节 | 延迟 |
|------|-------------|---------|------|
| 传统 read/write | 2 | 2000+ | 约 100us |
| sendfile | 0 | 0 | 约 10us |
| splice | 0 | 0 | 约 10us |

### 上下文切换开销

**系统调用开销**：

每次系统调用约消耗 1000-2000 CPU 周节：

- 用户态到内核态切换
- 参数验证
- 内核处理
- 返回用户态

**传统 vs 零拷贝**：

| 方式 | 系统调用次数 | 上下文切换开销 |
|------|-------------|---------------|
| 传统 read + write | 2 | 约 2000-4000 周节 |
| sendfile | 1 | 约 1000-2000 周节 |
| splice (两次) | 2 | 约 2000-4000 周节 |
| io_uring | 0（轮询模式） | 0 |

### 内存带宽分析

**内存带宽消耗**：

以 DDR4 3200 为例，理论带宽约 25.6 GB/s。

传统方式下，64KB 数据需要复制 4 次，消耗约 256KB 的带宽。

零拷贝方式下，64KB 数据仅需要 2 次 DMA 传输，消耗约 128KB 的带宽。

**高并发场景**：

假设 10,000 个并发连接，每个连接传输 1MB/s：

- 传统方式：10,000 * 4 * 1MB = 40 GB/s 带宽需求
- 零拷贝方式：10,000 * 2 * 1MB = 20 GB/s 带宽需求

零拷贝将内存带宽需求减半。

### 延迟分析

**传输延迟组成**：

```
总延迟 = 系统调用延迟 + 复制延迟 + DMA 传输延迟

传统方式:
- read() 系统调用: 约 10us
- 内核到用户复制: 约 50us (64KB)
- write() 系统调用: 约 10us
- 用户到内核复制: 约 50us
- DMA 传输: 约 10us
总计: 约 120us

零拷贝方式:
- sendfile() 系统调用: 约 10us
- DMA 传输: 约 10us
总计: 约 20us
```

**Prism 基准测试**：

| 场景 | 传统方式延迟 | PMR 优化延迟 | 改进 |
|------|-------------|-------------|------|
| 64KB 隧道转发 | 120us | 40us | 3x |
| 1MB 文件传输 | 2ms | 0.5ms | 4x |
| 10K 并发连接 | P99 500us | P99 100us | 5x |

### 缓存污染分析

**传统方式的缓存影响**：

数据复制过程会污染 CPU 缓存：
- 复制的数据进入缓存
- 原本有用的数据被挤出缓存
- 后续计算需要重新加载缓存

**零拷贝的缓存友好性**：

- 数据不经过 CPU，不污染缓存
- CPU 缓存保留有用的计算数据
- 计算性能不受数据传输影响

---

## 最佳实践

### 场景选择指南

**何时使用 sendfile**：

- 从文件发送到 Socket（静态文件服务）
- 数据不需要修改
- 文件大小适中（几 KB 到几 GB）

**何时使用 splice**：

- 管道间传输（如日志收集）
- Socket 到 Socket（代理转发）
- 需要灵活的中间处理

**何时使用 mmap**：

- 需要随机访问文件内容
- 多进程共享文件映射
- 文件内容需要多次访问

**何时使用 PMR/移动语义**：

- 应用内数据处理
- 容器间传递数据
- 协程内临时缓冲

### 缓冲区大小选择

**网络传输缓冲区**：

- 最小：4KB（减少系统调用次数）
- 推荐：16KB-64KB（平衡吞吐和内存）
- 最大：256KB-1MB（大吞吐场景，注意 TCP 窗口）

**Prism 默认配置**：

```json
{
  "buffer": {
    "size": 65536  // 64KB
  }
}
```

### 错误处理

**零拷贝的常见错误**：

```cpp
// sendfile 可能的错误
ssize_t sent = sendfile(socket_fd, file_fd, &offset, count);
if (sent < 0) {
  // EAGAIN: 非阻塞 Socket 缓冲区满
  // EINVAL: 文件不支持 mmap
  // ENOSYS: 内核不支持 sendfile
}
```

**回退策略**：

```cpp
void send_data(int socket, int file, size_t size) {
  // 尝试 sendfile
  if (sendfile(socket, file, nullptr, size) < 0) {
    // 失败时回退到 read/write
    fallback_send(socket, file, size);
  }
}
```

### 多平台兼容

**跨平台零拷贝实现**：

```cpp
#ifdef __linux__
  #define USE_SENDFILE 1
#elif defined(_WIN32)
  #define USE_TRANSMITFILE 1
#else
  #define USE_FALLBACK 1
#endif

void zero_copy_send(socket_t sock, file_t file, size_t size) {
#if USE_SENDFILE
  sendfile(sock, file, nullptr, size);
#elif USE_TRANSMITFILE
  TransmitFile(sock, file, size, 0, nullptr, nullptr, 0);
#else
  fallback_send(sock, file, size);
#endif
}
```

### 调试与监控

**监控零拷贝效果**：

```cpp
// 统计复制次数
struct stats {
  size_t sendfile_calls = 0;
  size_t fallback_reads = 0;
  size_t bytes_via_zero_copy = 0;
  size_t bytes_via_copy = 0;
};

void report_stats() {
  double ratio = stats.bytes_via_zero_copy / (double)total_bytes;
  std::cout << "零拷贝传输比例: " << ratio * 100 << "%\n";
}
```

---

## 常见问题

### Q1: 零拷贝总是更快吗？

**A**: 不一定。零拷贝有以下开销：
- sendfile/splice 的设置开销（系统调用）
- mmap 的页表和缺页中断开销
- 小数据量时，复制可能更快（缓存命中）

建议：小数据（<4KB）使用传统方式，大数据使用零拷贝。

### Q2: sendfile 可以用于加密数据吗？

**A**: 不能直接用于。sendfile 不经过用户空间，无法在传输过程中加密。解决方案：
- 使用 splice 到管道，在管道处理加密
- 或放弃零拷贝，使用用户空间加密

### Q3: mmap 和 sendfile 哪个更好？

**A**: 取决于场景：
- **mmap**：需要随机访问文件、多次访问同一数据
- **sendfile**：只需顺序发送文件、一次性传输

对于静态文件服务，sendfile 通常更优。

### Q4: splice 为什么需要管道？

**A**: splice 的设计使用管道作为中间层，原因是：
- 管道有现成的缓冲机制
- 管道支持阻塞/非阻塞语义
- 可以解耦源和目标的生命周期

### Q5: RDMA 在什么情况下值得使用？

**A**: RDMA 适合：
- 高吞吐量（>10 Gbps）
- 低延迟要求（<10us）
- 专用网络环境（InfiniBand、RoCE）

普通以太网环境下，sendfile/splice 已足够。

### Q6: 如何判断系统支持哪种零拷贝？

**A**: 通过系统调用返回值判断：
```c
// 检查 sendfile 支持
if (sendfile(sock, file, nullptr, 0) == -1 && errno == ENOSYS) {
  // 不支持 sendfile
}
```

或在编译时检查宏：
```c
#ifdef SYS_sendfile
  // 支持 sendfile
#endif
```

### Q7: 零拷贝对 SSD 和 HDD 有区别吗？

**A**: 有区别：
- **HDD**：磁盘读取延迟高，零拷贝收益更大
- **SSD**：磁盘读取延迟低，零拷贝收益较小但仍有意义

两者的 DMA 机制相同，零拷贝都能节省 CPU 复制。

### Q8: io_uring 与 sendfile 哪个更好？

**A**: io_uring 更适合高并发异步场景：
- **sendfile**：单次同步调用，适合简单场景
- **io_uring**：批量异步提交，适合高并发场景

Prism 等异步架构更适合 io_uring。

### Q9: Windows 有 splice 类似功能吗？

**A**: Windows 没有直接对应的 splice。可用：
- TransmitFile（类似 sendfile）
- Named Pipe + ReadFile/WriteFile
- Winsock 的 scatter-gather I/O

### Q10: 零拷贝如何与 SSL/TLS 配合？

**A**: SSL/TLS 加密必须在用户空间进行，数据需要复制到加密缓冲区：
- 先用 splice 从文件读取到管道
- 在用户空间从管道读取并加密
- 加密后数据写入 Socket

这打破了完全零拷贝，但 splice 部分仍有效。

---

## 参考资料

- [Zero-copy Networking](https://en.wikipedia.org/wiki/Zero-copy)
- [Linux sendfile](https://man7.org/linux/man-pages/man2/sendfile.2.html)
- [Linux splice](https://man7.org/linux/man-pages/man2/splice.2.html)
- [io_uring Documentation](https://kernel.dk/io_uring.pdf)
- [RDMA Overview](https://docs.nvidia.com/networking/display/RDMAAwareNETAPIV17/RDMA+Overview)

## 相关知识

- [[ref/memory/pmr|PMR 内存资源]] — 多态内存资源基础
- [[ref/memory/arena|Arena 分配器]] — 帧级线性分配
- [[core/channel/transport/transmission|传输层]] — Prism 隧道实现
- [[core/memory/container|PMR 容器别名]] — 容器类型定义