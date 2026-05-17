---
title: Go Concurrency Model
created: 2026-05-17
updated: 2026-05-17
layer: ref
tags: [programming, go, concurrency, goroutine, mihomo]
---

# Go 并发模型

> Go 语言并发模型（Goroutine、Channel），mihomo 应用参考。
> 最后更新：2026-05-17

---

## Goroutine

### 概念

轻量级线程，由 Go runtime 调度：

```go
// 启动 goroutine
go func() {
    fmt.Println("in goroutine")
}()

fmt.Println("in main")
```

### 特点

| 特性 | 说明 |
|------|------|
| 轻量 | 初始栈 2KB，可增长 |
| 调度 | runtime 合作式调度 |
| 成本 | 创建/切换成本低 |
| 数量 | 可创建百万级 |

### 与线程对比

| 特性 | Goroutine | 系统线程 |
|------|-----------|----------|
| 栈大小 | 2KB→动态 | 1-8MB 固定 |
| 调度 | runtime | 操作系统 |
| 创建成本 | 低 | 高 |
| 切换成本 | 低 | 高 |

---

## Channel

### 概念

goroutine 间通信管道：

```go
// 创建 channel
ch := make(chan int)
ch := make(chan int, 10)  // 带缓冲

// 发送
ch <- value

// 接收
value := <-ch
value, ok := <-ch  // 检查是否关闭
```

### 方向

```go
// 单向 channel
func send(ch chan<- int) {
    ch <- 42
}

func receive(ch <-chan int) {
    value := <-ch
}
```

### 关闭

```go
close(ch)

// 遍历
for value := range ch {
    // 处理
}
```

---

## Select

### 多路选择

```go
select {
case v := <-ch1:
    fmt.Println("from ch1:", v)
case v := <-ch2:
    fmt.Println("from ch2:", v)
case ch3 <- 42:
    fmt.Println("sent to ch3")
default:
    fmt.Println("no activity")
}
```

### 超时

```go
select {
case v := <-ch:
    process(v)
case <-time.After(5 * time.Second):
    fmt.Println("timeout")
}
```

---

## Context

### 用途

- 超时控制
- 取消信号
- 值传递

### 类型

```go
// 创建
ctx := context.Background()
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
ctx, cancel := context.WithCancel(ctx)
ctx := context.WithValue(ctx, "key", "value")

// 使用
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // 工作
        }
    }
}
```

---

## Sync 包

### Mutex

```go
var mu sync.Mutex

mu.Lock()
// 临界区
mu.Unlock()
```

### RWMutex

```go
var mu sync.RWMutex

mu.RLock()   // 读锁
// 读操作
mu.RUnlock()

mu.Lock()    // 写锁
// 写操作
mu.Unlock()
```

### WaitGroup

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        // 工作
    }()
}

wg.Wait()
```

### Once

```go
var once sync.Once

once.Do(func() {
    // 只执行一次
})
```

---

## Mihomo 并发应用

### 连接处理

mihomo 使用 goroutine 处理连接：

```go
// mihomo 连接处理（示意）
func handleConnection(conn net.Conn) {
    go func() {
        defer conn.Close()
        processTraffic(conn)
    }()
}
```

### 协议处理

```go
// 协议处理 goroutine
func (a *Adapter) HandleTCP(conn net.Conn) {
    go func() {
        tunnel := a.tunnelTCP(conn)
        relay(tunnel)
    }()
}
```

### UDP 处理

```go
// UDP NAT 表
type NatTable struct {
    mu sync.RWMutex
    table map[string]*UDPConn
}

func (t *NatTable) Get(key string) *UDPConn {
    t.mu.RLock()
    defer t.mu.RUnlock()
    return t.table[key]
}
```

---

## Go vs C++ 协程对比

| 特性 | Go Goroutine | C++ 协程 |
|------|--------------|----------|
| 语法 | `go func()` | `co_await` |
| 调度 | runtime 自动 | 用户控制 |
| 栈 | 动态增长 | 堆帧固定 |
| 通信 | Channel | 无内置 |
| 错误 | panic/recover | try/catch |
| 取消 | Context | 无内置 |

---

## Prism 参考

mihomo 的 Go 并发模型是 Prism C++ 协程设计的参考：

| Go 概念 | Prism 对应 | 详见 |
|---------|------------|------|
| Goroutine | C++20/23 协程 | [[cpp23-coroutine|C++23 协程]] |
| Channel | 无直接对应 | — |
| Context | 取消信号 | [[core/channel/overview|Channel]] |
| Mutex | std::mutex | [[ref/network/happy-eyeballs|Happy Eyeballs]] |

---

## 相关参考

- [[ref/mihomo/overview|Mihomo 总索引]] — mihomo 模块
- [[cpp23-coroutine|C++23 协程]] — Prism 异步模型
- [[boost-asio|Boost.Asio 协程]] — Prism 网络层

---

## 进一步阅读

- Go 并发: https://go.dev/doc/effective_go#concurrency
- Goroutine: https://go.dev/tour/concurrency/1
- Channel: https://go.dev/tour/concurrency/2