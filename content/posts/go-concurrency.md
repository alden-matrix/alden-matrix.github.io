---
title: "Go 基础篇：goroutine 和 channel 从使用到原理"
date: 2026-03-15
tags: ["Go", "Go基础", "并发", "goroutine", "channel", "源码分析"]
summary: "Go 的并发是它最大的卖点，但 goroutine 为什么这么轻量、channel 底层是怎么实现的、什么时候用 channel 什么时候用 mutex，很多人说不清楚。这篇从 runtime 源码出发，把 Go 并发讲透。"
draft: false
---

## goroutine 到底有多轻量

先感受一下数字：

| 对比项 | OS 线程 | goroutine |
|-------|---------|-----------|
| 初始栈大小 | 1~8MB（固定） | **2KB**（动态增长） |
| 创建开销 | 微秒级（系统调用） | **纳秒级**（用户态） |
| 切换开销 | 微秒级（内核态切换） | **纳秒级**（只保存几个寄存器） |
| 百万个占用 | 几 TB 内存，系统撑不住 | **约 2GB**，正常跑 |

一个 OS 线程的栈是 MB 级别的，而且不能缩小。goroutine 的初始栈只有 2KB，不够了自动翻倍增长，用完了 GC 时还会缩回去。

这个差距不是优化出来的，是设计上就不一样——goroutine 是用户态调度，不需要陷入内核。

---

## goroutine 的创建过程

`go func()` 这行代码，编译器会翻译成 `runtime.newproc` 调用。流程很简洁：

```
1. 先从当前 P 的空闲列表里找有没有用完的 g 可以复用
   → 有：直接拿来重新初始化（省掉内存分配）
   → 没有：malg(2048) 分配一个新 g，2KB 栈

2. 设置 g 的寄存器上下文：pc 指向目标函数，sp 指向栈顶

3. 放入当前 P 的本地运行队列

4. 如果有空闲的 P 没人用，唤醒一个 M 来执行
```

关键点：goroutine 不是每次都分配新内存。执行完的 goroutine 会被回收到空闲列表，下次 `go func()` 直接复用。这就是为什么高频创建 goroutine 的开销很低。

---

## GMP 调度模型

Go 的调度器由三个角色组成：

- **G（Goroutine）**：你写的 `go func()`，包含栈、寄存器上下文、状态
- **M（Machine）**：OS 线程，真正执行代码的载体
- **P（Processor）**：逻辑处理器，M 必须绑定一个 P 才能执行 Go 代码

```
           ┌──────────────────────┐
           │    全局运行队列        │
           │  （溢出和负载均衡用）   │
           └──────────┬───────────┘
                      │
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │    P0    │   │    P1    │   │    P2    │
   │ 本地队列  │   │ 本地队列  │   │ 本地队列  │
   │ [256个槽] │   │ [256个槽] │   │ [256个槽] │
   └────┬─────┘   └────┬─────┘   └────┬─────┘
        │              │              │
   ┌────┴─────┐   ┌────┴─────┐   ┌────┴─────┐
   │   M0     │   │   M1     │   │   M2     │
   │ (OS线程)  │   │ (OS线程)  │   │ (OS线程)  │
   │  运行 G   │   │  运行 G   │   │  运行 G   │
   └──────────┘   └──────────┘   └──────────┘
```

**为什么需要 P？** 如果只有 G 和 M，那所有 goroutine 共享一个全局队列，每次取任务都要加锁，M 越多竞争越激烈。P 的作用就是给每个 M 一个私有的本地队列，本地操作不需要加锁。

**GOMAXPROCS** 控制的就是 P 的数量，默认等于 CPU 核数。8 核机器有 8 个 P，意味着最多 8 个 goroutine 同时执行。

### work stealing：忙的帮闲的

当一个 P 的本地队列空了，M 不会闲着，而是去**偷**别的 P 的工作：

```
M 找工作的顺序（findRunnable 函数）：
1. 每 61 次调度检查一次全局队列（防止全局队列被饿死）
2. 检查自己 P 的本地队列
3. 检查全局队列
4. 检查网络轮询器（有没有 I/O 就绪的 goroutine）
5. 随机选一个 P，偷它本地队列一半的 G
6. 以上全部失败 → M 休眠
```

偷取时随机选 P，最多尝试 4 轮。这保证了 goroutine 在多个 P 之间均匀分布，不会出现有的核忙死、有的核闲死。

### 抢占：不让一个 goroutine 霸占 CPU

**协作式抢占（Go 1.2+）**

编译器在每个函数入口插入一段栈检查代码。运行时想抢占某个 G 时，把它的 `stackguard0` 设为一个特殊值（`stackPreempt`）。下次 G 调用函数时栈检查失败，进入调度器让出 CPU。

问题：如果 G 在执行一个没有函数调用的紧密循环（比如 `for { i++ }`），栈检查永远不会触发，这个 G 就永远不会被抢占。

**异步信号抢占（Go 1.14+）**

解决方案：运行时直接给目标 M 发送 **SIGURG 信号**。信号处理器检查当前是否在安全点，如果是，就往 G 的栈上注入一个 `asyncPreempt` 函数调用，G 恢复执行后会立即进入调度器。

为什么选 SIGURG？因为几乎没有应用程序依赖这个信号，调试器也默认透传它，不会影响 GDB 使用。

**Go 1.14 之后，即使是 `for {}` 死循环也能被抢占了。**

---

## goroutine 的栈管理

goroutine 的栈不是固定大小，而是按需增长、按需收缩。

### 增长：连续栈

当栈空间不足时（函数入口的栈检查触发），`runtime.newstack` 执行：

```
1. 分配一个 2 倍大小的新栈
2. 把旧栈的内容完整拷贝过去
3. 更新所有指向旧栈的指针（这是最复杂的部分）
4. 释放旧栈
```

这叫**连续栈**（contiguous stack），Go 1.4 引入。之前用的是分段栈（segmented stack），但有个 "hot split" 问题：如果函数调用恰好在栈边界反复进出，每次都要分配/释放栈段，性能很差。连续栈彻底解决了这个问题。

### 收缩：GC 时自动处理

GC 扫描到一个 goroutine 时，如果发现它的栈使用量不到 1/4，就会把栈缩小为一半（最小 2KB）。所以大量闲置的 goroutine 不会浪费内存。

---

## channel 的内存结构

channel 在 runtime 中是 `hchan` 结构体（`runtime/chan.go`）：

```go
type hchan struct {
    qcount   uint           // 缓冲区中当前的元素数量
    dataqsiz uint           // 缓冲区容量（make 时指定的 N）
    buf      unsafe.Pointer // 环形缓冲区指针
    elemsize uint16         // 单个元素大小
    closed   uint32         // 是否已关闭
    sendx    uint           // 下一个写入位置
    recvx    uint           // 下一个读取位置
    recvq    waitq          // 等待接收的 goroutine 队列
    sendq    waitq          // 等待发送的 goroutine 队列
    lock     mutex          // 互斥锁
}
```

画出来：

```
ch := make(chan int, 4)

  hchan
  ┌──────────────────────────────────┐
  │ qcount   = 0                     │
  │ dataqsiz = 4                     │
  │ buf ──→ [ _ ][ _ ][ _ ][ _ ]    │  ← 环形缓冲区
  │ sendx = 0, recvx = 0            │
  │ recvq → (空)                     │
  │ sendq → (空)                     │
  │ lock                             │
  └──────────────────────────────────┘
```

**无缓冲 channel**（`make(chan int)`）：`dataqsiz = 0`，`buf = nil`，没有缓冲区，发送方必须等到接收方准备好。

**有缓冲 channel**（`make(chan int, N)`）：`buf` 指向 N 个元素大小的环形数组，`sendx` 和 `recvx` 分别追踪写入和读取位置。

---

## channel 的发送和接收

### 发送（ch <- v）

加锁后，按优先级走三条路径：

**路径一：有 goroutine 在等着接收**（recvq 不为空）

直接把数据拷贝到等待者的栈上，然后唤醒它。不经过缓冲区。

```
发送者的 v ──memmove──→ 接收者栈上的变量
```

这是 Go 运行时中**唯一一处一个 goroutine 直接写另一个 goroutine 栈**的操作。对于无缓冲 channel，数据从发送者栈直接到接收者栈，零中间拷贝。

**路径二：缓冲区没满**

把数据拷贝到 `buf[sendx]`，`sendx` 前进一步，解锁返回。

**路径三：缓冲区满了（或无缓冲）**

创建一个 `sudog` 结构体，记录当前 goroutine 和待发送的数据地址，挂到 `sendq` 队列，然后 `gopark` 阻塞自己。等有人来接收时才会被唤醒。

### 接收（v := <-ch）

逻辑是镜像的：

1. 有发送者在等 → 直接从发送者栈拷贝（无缓冲），或从 buf 取数据并把发送者的数据补入 buf（有缓冲）
2. 缓冲区有数据 → 从 `buf[recvx]` 取
3. 都没有 → 挂到 `recvq` 阻塞

### 关闭（close(ch)）

加锁，设置 `closed = 1`，唤醒 `recvq` 和 `sendq` 中所有等待者。接收者收到零值，发送者 panic。

**记住这几条：**
- 向已关闭的 channel 发送 → **panic**
- 从已关闭的 channel 接收 → 返回零值 + `false`
- 关闭一个 nil channel → **panic**
- 关闭一个已关闭的 channel → **panic**

---

## select 的实现

```go
select {
case v := <-ch1:
    // ...
case ch2 <- x:
    // ...
default:
    // ...
}
```

`select` 在运行时由 `selectgo` 函数实现，分三步：

**第一步：随机打乱 case 顺序**

保证公平性。如果不打乱，第一个 case 永远优先被检查，多个 channel 同时就绪时总是选第一个。

**第二步：按 channel 地址排序加锁**

select 可能涉及多个 channel，要给它们都加锁。为了避免死锁，所有 select 都按 channel 的内存地址从小到大加锁。

**第三步：检查是否有就绪的 case**

按随机顺序遍历所有 case，看有没有能立即执行的。有就执行并返回。都不行就：
- 有 `default` → 执行 default
- 没有 `default` → 把当前 goroutine 注册到**所有**相关 channel 的等待队列，然后阻塞。被任一 channel 唤醒后，从其他所有 channel 的等待队列中移除自己。

---

## 常用并发模式

### worker pool

最常用的模式：固定数量的 goroutine 消费任务队列。

```go
func processJobs(jobs []Job, workerCount int) []Result {
    jobCh := make(chan Job, len(jobs))
    resultCh := make(chan Result, len(jobs))

    // 启动 worker
    var wg sync.WaitGroup
    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobCh {
                resultCh <- job.Process()
            }
        }()
    }

    // 发送任务
    for _, job := range jobs {
        jobCh <- job
    }
    close(jobCh)

    // 等待完成并收集结果
    wg.Wait()
    close(resultCh)

    var results []Result
    for r := range resultCh {
        results = append(results, r)
    }
    return results
}
```

为什么不直接 `go func()` 每个任务一个 goroutine？小任务可以，但如果任务涉及 I/O（数据库、HTTP 请求），无限制创建 goroutine 会打满连接池。worker pool 限制了并发度。

### errgroup：并发执行 + 错误收集

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]Response, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Response, len(urls))

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            resp, err := httpGet(ctx, url)
            if err != nil {
                return err // 第一个错误会触发 ctx 取消
            }
            results[i] = resp
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

`errgroup` 做了三件事：等待所有 goroutine 完成、收集第一个错误、通过 context 通知其他 goroutine 取消。自己写要好几十行，用它一行搞定。

### context 取消传播

```go
func longRunningTask(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            // 上游取消了，清理资源后退出
            return ctx.Err()
        default:
            // 执行一批工作
            if err := doBatch(); err != nil {
                return err
            }
        }
    }
}
```

context 是 Go 并发的基础设施。任何可能被取消的操作（HTTP 请求、数据库查询、定时任务）都应该接收 `context.Context` 参数，并检查 `ctx.Done()`。

---

## 并发陷阱

### goroutine 泄漏

goroutine 泄漏就是创建了但永远不会退出，最终 goroutine 数量无限增长，内存耗尽。

**最常见的原因：channel 没人读/写**

```go
// ❌ 泄漏：如果 process 返回错误提前退出，这个 goroutine 永远阻塞
func handle(ctx context.Context) error {
    ch := make(chan Result)
    go func() {
        ch <- doExpensiveWork() // 如果没人读 ch，这里永远阻塞
    }()

    if err := validate(); err != nil {
        return err // 提前退出了，没有读 ch
    }
    result := <-ch
    return process(result)
}

// ✅ 修复：用 select + context 保证 goroutine 能退出
func handle(ctx context.Context) error {
    ch := make(chan Result, 1) // 缓冲为 1，即使没人读也不阻塞
    go func() {
        ch <- doExpensiveWork()
    }()

    if err := validate(); err != nil {
        return err
    }
    result := <-ch
    return process(result)
}
```

**检测方法：**
- `runtime.NumGoroutine()` 加到监控里，持续增长就有泄漏
- 测试用 `go.uber.org/goleak`，测试结束时验证 goroutine 全部退出
- `pprof` 的 goroutine profile 看哪些 goroutine 卡在哪里

### data race

```go
// ❌ 并发写 map 会 fatal（不是 panic，recover 不了）
m := make(map[string]int)
for i := 0; i < 10; i++ {
    go func() {
        m["key"] = i // fatal error: concurrent map writes
    }()
}

// ✅ 用 sync.Map 或加锁
var mu sync.Mutex
for i := 0; i < 10; i++ {
    go func() {
        mu.Lock()
        m["key"] = i
        mu.Unlock()
    }()
}
```

**一定要用 `go test -race` 跑测试**。race detector 能发现绝大多数数据竞争，没有误报（报了就一定有问题），代价是运行速度慢 5~10 倍。在 CI 里开一个 race 测试 job，上线前必过。

### channel vs mutex：什么时候用哪个

Go 有句名言："Do not communicate by sharing memory; instead, share memory by communicating." 但这不是说什么都用 channel。

| 场景 | 用什么 | 原因 |
|------|--------|------|
| 传递数据的所有权 | channel | 数据发出去就不再持有 |
| 任务分发 / worker pool | channel | 天然的生产者消费者模型 |
| 等待异步结果 | channel | 一次性通知 |
| pipeline / fan-out / fan-in | channel | 数据流 |
| 保护共享状态（缓存、配置） | **mutex** | 锁保护更直觉更快 |
| 简单计数器 | **sync/atomic** | 一条指令，最快 |
| 等待一组 goroutine 完成 | **sync.WaitGroup** | 比 channel 计数更简洁 |

经验法则：**如果你发现自己用 channel 在做"加锁"的事（比如缓冲为 1 的 channel 当互斥锁），那直接用 mutex 更清晰。**

---

## 总结

| 知识点 | 记住这一句 |
|--------|-----------|
| goroutine 开销 | 初始 2KB 栈 + 纳秒级创建，百万级并发没问题 |
| GMP | G 是协程，M 是线程，P 是调度资源；P 数量 = GOMAXPROCS |
| work stealing | P 空了就随机偷别人一半的 G，保证负载均衡 |
| 抢占 | Go 1.14 用 SIGURG 信号实现异步抢占，死循环也能被打断 |
| 栈管理 | 连续栈，不够了翻倍，GC 时用得少会缩回来 |
| channel 结构 | 环形缓冲区 + 两个等待队列 + 一把锁 |
| 直接拷贝 | 有等待者时数据直接从一个 goroutine 栈拷贝到另一个，零中间缓冲 |
| select | 随机打乱保证公平，按地址加锁避免死锁 |
| goroutine 泄漏 | 最常见原因是 channel 没人读写，用缓冲或 context 保护 |
| channel vs mutex | channel 用于通信和协调，mutex 用于保护共享状态 |
