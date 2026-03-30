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

OS 线程的栈在创建时就分配了 MB 级别的内存，而且大小固定。goroutine 的初始栈只有 2KB（`runtime/stack.go` 中的 `_StackMin = 2048`），不够了自动翻倍增长，GC 时发现使用量不到 1/4 还会缩回来。

**这个差距不是优化出来的，是设计上就不一样**——goroutine 是用户态调度，创建和切换都不需要陷入内核。

### goroutine 的创建过程

`go func()` 编译器会翻译成 `runtime.newproc` 调用：

```
1. 从当前 P 的空闲列表（gFree）找有没有执行完的 g 可以复用
   → 有：直接拿来重新初始化，省掉内存分配
   → 没有：malg(2048) 分配新的 g，初始 2KB 栈

2. 设置 g 的寄存器上下文（sched 字段）：pc 指向目标函数，sp 指向栈顶

3. 放入当前 P 的本地运行队列（优先放入 runnext 槽位）

4. 如果有空闲的 P 没有关联 M，唤醒一个 M 来执行
```

**关键点**：goroutine 不是每次都分配新内存。执行完的 goroutine 状态变为 `_Gdead`，回收到 P 的空闲列表，下次 `go func()` 直接复用。所以高频创建 goroutine 的开销很低。

### 栈的动态伸缩

goroutine 的栈不是固定大小，而是按需伸缩：

**增长**：编译器在每个函数入口插入栈检查。栈空间不足时，`runtime.newstack` 分配一个 2 倍大小的新栈，把旧栈内容完整拷贝过去，更新所有指向旧栈的指针，释放旧栈。这叫**连续栈**（Go 1.3 引入，Go 1.4 进一步将默认栈从 8KB 降到 2KB），彻底解决了之前分段栈的 "hot split" 性能问题。

**收缩**：GC 扫描时，如果 goroutine 栈使用量不到当前栈大小的 1/4，就缩小为原来的一半（最小 2KB）。所以大量闲置 goroutine 不会浪费内存。

---

## GMP 调度模型

Go 的调度器由三个角色组成：

- **G（Goroutine）**：用户协程，包含栈、寄存器上下文、状态
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

### 为什么需要 P

这是理解 GMP 的关键。

如果只有 G 和 M（Go 1.0 的设计），所有 goroutine 共享一个全局队列，每次取任务都要加锁。M 越多竞争越激烈，8 核机器上 8 个 M 抢一把锁，性能瓶颈明显。

引入 P 后，每个 P 有自己的本地队列（`runtime/runtime2.go` 中的 `runq [256]guintptr`，固定 256 个槽位的环形数组），M 绑定 P 后从本地队列取 G，**不需要加锁**。全局队列只在本地队列溢出或为空时才用到。

**GOMAXPROCS** 控制的就是 P 的数量，默认等于 CPU 核数。8 核机器有 8 个 P，最多 8 个 goroutine 同时执行用户代码。注意：被阻塞在系统调用中的 M 不占 P，所以实际 M 的数量可能超过 GOMAXPROCS。

### work stealing

当一个 P 的本地队列空了，M 不会闲着，而是去**偷**别人的工作。`runtime/proc.go` 中的 `findRunnable` 函数定义了找工作的完整顺序：

```
1. 每 61 次调度检查一次全局队列（防止全局队列被饿死）
2. 检查自己 P 的本地队列
3. 检查全局队列
4. 检查网络轮询器（有没有 I/O 就绪的 goroutine）
5. 随机选一个 P，偷它本地队列一半的 G（最多尝试 4 轮）
6. 以上全部失败 → M 进入休眠
```

为什么是"每 61 次"？61 是一个质数，用质数做间隔可以避免多个 P 以相同节奏同时检查全局队列导致锁争用。

为什么偷一半而不是偷一个？偷一半可以减少偷取的次数，一次偷取就能让自己忙一阵子，而不是偷一个处理完又要偷。

### 抢占

**协作式抢占（Go 1.2+）**

编译器在每个函数入口插入栈检查代码（同时也用于栈增长检测）。运行时想抢占某个 G 时，把它的 `stackguard0` 设为特殊值 `stackPreempt`（定义为 `^uintptr(0) - 1313`，一个极大的地址值，任何栈检查都会失败）。下次 G 调用任何函数时栈检查失败，进入 `morestack` → `newstack`，发现是抢占请求而不是真的栈不足，于是让出 CPU。

**问题**：如果 G 在执行一个没有函数调用的紧密循环（`for { i++ }`），栈检查永远不会触发，这个 G 就永远不会被抢占。在 Go 1.14 之前，一个 `for {}` 能把整个 P 卡死。

**异步信号抢占（Go 1.14+）**

运行时直接给目标 M 发送 **SIGURG 信号**。信号处理器（`doSigPreempt`）检查当前 G 是否在安全点（能安全扫描栈、未持有运行时锁、栈空间充足），如果是，往 G 的栈上注入 `asyncPreempt` 函数调用。G 恢复执行后立即进入调度器让出 CPU。

为什么选 SIGURG？三个原因：几乎没有应用程序依赖它，调试器（GDB/LLDB）默认不拦截它，libc 内部不使用它。

**Go 1.14 之后，即使是 `for {}` 死循环也能被抢占。**

---

## channel 的内存结构

channel 在 runtime 中是 `hchan` 结构体（`runtime/chan.go`），以下是核心字段（省略了 `timer` 等与本文无关的字段）：

```go
type hchan struct {
    qcount   uint           // 缓冲区中当前的元素数量
    dataqsiz uint           // 缓冲区容量（make(chan T, N) 中的 N）
    buf      unsafe.Pointer // 指向环形缓冲区
    elemsize uint16         // 单个元素大小
    closed   uint32         // 是否已关闭（0 或 1）
    elemtype *_type         // 元素类型信息（GC 用）
    sendx    uint           // 下一个写入位置
    recvx    uint           // 下一个读取位置
    recvq    waitq          // 等待接收的 goroutine 队列（双向链表）
    sendq    waitq          // 等待发送的 goroutine 队列（双向链表）
    lock     mutex          // 保护所有字段的互斥锁
}
```

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

**无缓冲 channel**（`make(chan int)`）：`dataqsiz = 0`，`buf` 不分配。发送方必须等到接收方准备好才能完成操作。

**有缓冲 channel**（`make(chan int, N)`）：`buf` 指向 N 个元素的环形数组，`sendx` 和 `recvx` 作为写入和读取下标，到达 `dataqsiz` 时归零形成环形。

---

## channel 的发送和接收

### 发送（ch <- v）

加锁后，按优先级走三条路径：

**路径一：有 goroutine 在等着接收**（`recvq` 不为空）

从 `recvq` 取出等待者，把数据**直接拷贝到接收者的栈上**（通过 `sendDirect` 函数调用 `memmove`），然后唤醒接收者。不经过缓冲区。

```
发送者栈上的 v ──memmove──→ 接收者栈上的目标变量
```

这是 Go 运行时中**唯一一处一个 goroutine 直接写另一个 goroutine 栈**的操作（runtime 源码注释原文："the only place where one goroutine writes to the stack of another"）。对于无缓冲 channel，数据始终走这条路径，零中间缓冲。

**路径二：缓冲区没满**（`qcount < dataqsiz`）

把数据拷贝到 `buf[sendx]`，`sendx` 前进一步（到达 `dataqsiz` 时归零），`qcount++`，解锁返回。

**路径三：缓冲区满了（或无缓冲且没有接收者）**

创建一个 `sudog` 结构体（goroutine 在等待队列中的表示），记录当前 goroutine 和待发送数据的地址，挂到 `sendq` 队列尾部，然后 `gopark()` 把自己挂起。等有人来接收时才会被唤醒。

### 接收（v := <-ch）

逻辑是发送的镜像，但有一个细节不同：

**路径一：有发送者在等**（`sendq` 不为空）

分两种情况：
- **无缓冲 channel**：直接从发送者栈拷贝到接收者（`recvDirect`）
- **有缓冲 channel**（缓冲区一定是满的，否则发送者不会阻塞）：从 `buf[recvx]` 取数据给接收者，然后把发送者的数据写入 `buf[sendx]`（腾出一个位置），唤醒发送者

**路径二**：缓冲区有数据 → 从 `buf[recvx]` 取

**路径三**：都没有 → 挂到 `recvq` 阻塞

### 关闭（close(ch)）

加锁，设置 `closed = 1`，遍历 `recvq` 和 `sendq` 唤醒所有等待者。然后**释放锁之后**才调用 `goready` 唤醒这些 goroutine（先释放锁再唤醒，避免被唤醒的 goroutine 立刻尝试加锁导致争用）。

**记住这几条规则：**

| 操作 | 结果 |
|------|------|
| 向已关闭的 channel 发送 | **panic** |
| 从已关闭的 channel 接收 | 返回零值，第二个返回值为 `false` |
| 关闭 nil channel | **panic** |
| 关闭已关闭的 channel | **panic** |
| 向 nil channel 发送 | **永久阻塞** |
| 从 nil channel 接收 | **永久阻塞** |

最后两条容易忘：nil channel 的发送和接收都会永久阻塞，不是 panic。这个特性在 select 中有用——把某个 case 的 channel 设为 nil 可以"禁用"这个 case。

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

`select` 在运行时由 `runtime/select.go` 中的 `selectgo` 函数实现，分三步：

**第一步：随机打乱 case 顺序**（`pollorder`）

保证公平性。如果不打乱，第一个 case 永远优先被检查。当多个 channel 同时就绪时，哪个被选中应该是随机的。

**第二步：按 channel 地址排序确定加锁顺序**（`lockorder`）

select 可能涉及多个 channel，要给它们都加锁。如果两个 goroutine 的 select 涉及相同的两个 channel 但加锁顺序不同，就会死锁。所以所有 select 统一按 channel 的内存地址从小到大加锁。

**第三步：按随机顺序检查是否有就绪的 case**

遍历 `pollorder`，看有没有能立即执行的操作。有就执行并返回。都不行则：
- 有 `default` → 执行 default 返回
- 没有 `default` → 为每个 case 创建一个 `sudog`，注册到对应 channel 的等待队列，然后 `gopark` 阻塞。被任一 channel 唤醒后，从**所有其他 channel** 的等待队列中移除自己，返回被唤醒的那个 case 的结果

---

## 常用并发模式

### worker pool

固定数量的 goroutine 消费共享的任务 channel：

```go
func processJobs(ctx context.Context, jobs []Job, workerCount int) ([]Result, error) {
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
    close(jobCh) // 关闭后 worker 的 range 会退出

    // 等待所有 worker 完成
    wg.Wait()
    close(resultCh)

    results := make([]Result, 0, len(jobs))
    for r := range resultCh {
        results = append(results, r)
    }
    return results, nil
}
```

**什么时候用 worker pool，什么时候直接 `go func()`？**

如果任务是纯 CPU 计算（比如图片处理），goroutine 开销很低，直接 `go func()` 就行，Go 调度器会自动分配到多核上。

但如果任务涉及 I/O（数据库查询、HTTP 请求、文件操作），无限制创建 goroutine 会打满连接池或文件描述符。worker pool 的核心价值是**限制并发度**，不是"节省 goroutine 创建开销"。

### errgroup

`golang.org/x/sync/errgroup` 解决的是"并发执行多个任务，任一失败时通知其他任务取消"的问题：

```go
func fetchAll(ctx context.Context, urls []string) ([]Response, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Response, len(urls))

    for i, url := range urls {
        i, url := i, url // Go 1.22 之前需要这行来避免闭包捕获问题
        g.Go(func() error {
            resp, err := httpGet(ctx, url)
            if err != nil {
                return err
            }
            results[i] = resp // 每个 goroutine 写不同的下标，不需要加锁
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

`errgroup` 内部做了三件事：用 `sync.WaitGroup` 等待所有 goroutine、收集第一个返回的 error、通过 context 取消通知其他 goroutine。

注意 `results[i] = resp` 这行——每个 goroutine 写的是不同下标，所以不存在数据竞争，不需要加锁。但如果改成 `results = append(results, resp)` 就有竞争了，因为 append 可能修改 slice header。

---

## 并发陷阱

### goroutine 泄漏

goroutine 泄漏就是创建了但永远不会退出，goroutine 数量持续增长，最终内存耗尽。

**最常见的原因：channel 没人读/写**

```go
// ❌ 泄漏
func handle(ctx context.Context) error {
    ch := make(chan Result)
    go func() {
        ch <- doExpensiveWork() // 如果没人读 ch，这里永远阻塞
    }()

    if err := validate(); err != nil {
        return err // 提前退出了，没有读 ch，goroutine 永远泄漏
    }
    return process(<-ch)
}
```

有人会说"把 ch 改成缓冲为 1 的"来修。缓冲为 1 确实能让 goroutine 发送成功后退出，但 `doExpensiveWork()` 本身可能执行很久（比如一个 HTTP 请求），函数已经返回了但 goroutine 还在后台跑着白白消耗资源。

**正确做法：用 context 让 goroutine 能感知到取消**

```go
// ✅ 正确
func handle(ctx context.Context) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel() // 函数退出时取消 context

    ch := make(chan Result, 1)
    go func() {
        result, err := doExpensiveWork(ctx) // 传入 ctx，内部检查 ctx.Done()
        if err != nil {
            return // ctx 被取消时，doExpensiveWork 应该尽快返回
        }
        ch <- result
    }()

    if err := validate(); err != nil {
        return err // cancel() 被 defer 调用，goroutine 通过 ctx.Done() 感知到退出
    }

    select {
    case result := <-ch:
        return process(result)
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

关键改动：
1. `doExpensiveWork` 接收 `ctx` 参数，内部通过 `ctx.Done()` 检查取消
2. `defer cancel()` 保证函数退出时通知 goroutine
3. 缓冲为 1 防止 goroutine 发送时阻塞（万一 select 没走 ch 那个 case）

**检测方法：**
- `runtime.NumGoroutine()` 加到监控指标里，持续增长就有泄漏
- 测试用 `go.uber.org/goleak`，在 `TestMain` 里验证测试结束后没有多余 goroutine
- `pprof` 的 goroutine profile 看哪些 goroutine 卡在哪

### data race

Go 的 map 不支持并发读写，并发写会直接触发 `fatal error`（不是 panic，recover 不了）：

```go
// ❌ fatal error: concurrent map writes
func main() {
    m := make(map[string]int)
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        i := i // Go 1.22 之前需要，避免闭包捕获同一个 i
        go func() {
            defer wg.Done()
            m[fmt.Sprintf("key_%d", i)] = i // 并发写 map
        }()
    }
    wg.Wait()
}
```

注意这个例子里有两个并发问题：
1. **map 并发写**——多个 goroutine 同时写 map 会 fatal
2. **闭包捕获循环变量**（Go 1.22 之前）——如果不加 `i := i`，所有 goroutine 共享同一个 `i`，值都是循环结束后的 10

**修复方法取决于场景：**

```go
// 方案一：sync.Mutex（最通用）
var mu sync.Mutex
for i := 0; i < 10; i++ {
    i := i
    go func() {
        mu.Lock()
        m[fmt.Sprintf("key_%d", i)] = i
        mu.Unlock()
    }()
}

// 方案二：sync.Map（读多写少的场景更高效）
var sm sync.Map
for i := 0; i < 10; i++ {
    i := i
    go func() {
        sm.Store(fmt.Sprintf("key_%d", i), i)
    }()
}
```

`sync.Map` 不是 `map` 的并发安全版替代品。它针对两种场景做了优化：**key 稳定后几乎只读**，或者**不同 goroutine 操作不同的 key**。如果是频繁读写相同的 key，普通 map + Mutex 反而更快。

**`go test -race` 必须用**。race detector 能检测到绝大多数数据竞争，报了就一定有问题（零误报）。代价是运行速度慢 5~10 倍、内存占用增加 5~10 倍，所以不适合在生产环境开，但 CI 里必须有一个 race 测试。

### channel vs mutex

Go 有句名言："Do not communicate by sharing memory; instead, share memory by communicating." 但这不是说什么都用 channel。

| 场景 | 推荐 | 原因 |
|------|------|------|
| 传递数据所有权 | channel | 发出去就不再持有，自然避免竞争 |
| 任务分发 / worker pool | channel | 天然的生产者消费者模型 |
| 等待异步结果 | channel | 一次性通知 |
| pipeline / fan-out / fan-in | channel | 数据流 |
| 保护共享状态（缓存、计数器） | **sync.Mutex** | 更直觉，性能更好 |
| 简单的原子操作（计数、标志位） | **sync/atomic** | 一条 CPU 指令，最快 |
| 等待一组 goroutine 完成 | **sync.WaitGroup** | 比 channel 计数更简洁 |

**经验法则**：如果你在用 channel 做"保护共享状态"的事（比如缓冲为 1 的 channel 当互斥锁），那直接用 mutex 更清晰更快。channel 的开销比 mutex 大得多——channel 操作涉及加锁、内存拷贝、goroutine 调度，而 mutex 在无争用时只是一个原子操作。

反过来，如果你在用 mutex + condition variable 来协调多个 goroutine 的执行顺序，那大概率用 channel 写出来更简洁。

---

## 总结

| 知识点 | 记住这一句 |
|--------|-----------|
| goroutine 开销 | 初始 2KB 栈 + 纳秒级创建，执行完回收复用 |
| GMP | G 是协程，M 是线程，P 是调度资源；P 的本地队列无锁操作 |
| 为什么要 P | 消除全局队列的锁竞争，每个 M 从自己绑定的 P 取任务 |
| work stealing | P 空了就随机偷别人一半的 G，每 61 次调度查一次全局队列 |
| 抢占 | Go 1.14 用 SIGURG 信号实现异步抢占，死循环也能被打断 |
| 栈管理 | 连续栈，不够翻倍增长，GC 时用不到 1/4 会缩回来 |
| channel 结构 | 环形缓冲区 + 两个等待队列（sendq/recvq）+ 一把锁 |
| 发送优化 | 有等待的接收者时，数据直接从发送者栈拷贝到接收者栈 |
| nil channel | 发送和接收都永久阻塞（不是 panic） |
| select | 随机打乱保证公平，按地址排序加锁避免死锁 |
| goroutine 泄漏 | 用 context 让 goroutine 感知取消，不要只靠缓冲 channel |
| data race | map 并发写是 fatal 不是 panic，`go test -race` 必须开 |
| channel vs mutex | channel 用于通信协调，mutex 用于保护状态，不要混用 |
