---
title: "Go 基础篇：defer 不是你以为的那么简单 —— 从 50ns 到 6ns 的三次进化"
date: 2026-03-10
tags: ["Go", "Go基础", "defer", "panic", "recover", "源码分析"]
summary: "defer 谁都会用，但它在 Go 1.12 到 1.14 之间经历了三次底层重写，性能从 50ns 降到 6ns。这篇从源码讲清楚 defer 的三种实现、闭包陷阱、以及 panic/recover 的真实工作原理。"
draft: false
---

## defer 的基本规则

先快速过一遍基础，后面的内容都建立在这上面。

### 规则一：LIFO，后进先出

```go
func main() {
    defer fmt.Println("first")
    defer fmt.Println("second")
    defer fmt.Println("third")
}
// 输出：
// third
// second
// first
```

每个 goroutine 内部维护一个 defer 链表，新的 defer 插入链表头部，执行时从头开始遍历。自然就是后进先出。

### 规则二：参数在 defer 声明时就已求值

```go
func main() {
    i := 0
    defer fmt.Println(i) // i 在这一刻被复制了，值为 0
    i = 100
}
// 输出：0，不是 100
```

defer 会把参数值**立即复制**一份保存起来，后续对原变量的修改不会影响已保存的值。

### 规则三：闭包捕获的是变量本身，不是值

```go
func main() {
    i := 0
    defer func() {
        fmt.Println(i) // 闭包持有 i 的引用
    }()
    i = 100
}
// 输出：100
```

闭包捕获的是变量的**地址**，执行时读取的是最新值。这跟规则二不矛盾——规则二是值传递参数，规则三是闭包引用变量。

### 规则四：defer 能修改命名返回值

```go
func foo() (result int) {
    defer func() {
        result++
    }()
    return 0
}
// 返回 1，不是 0
```

这是因为 `return 0` 在编译器眼里是两步：先把 0 赋值给 `result`，再执行 defer 链，最后才真正返回。defer 闭包修改的是栈帧上的 `result` 变量，所以能改变返回值。

**注意**：只有命名返回值才会被 defer 修改。非命名返回值的 return 会把结果放到调用方无法访问的位置，defer 改不了。

---

## defer 的三次底层进化

上面的规则只是"怎么用"。下面讲"怎么实现的"——defer 在 Go 1.12 到 1.14 之间经历了三次重写，性能提升了将近 10 倍。

### Go 1.12 及之前：堆分配

每次遇到 `defer` 语句，运行时调用 `runtime.deferproc`，在**堆上**分配一个 `_defer` 结构体，挂到 goroutine 的 defer 链表头部。

```
defer fmt.Println("hello")

→ runtime.deferproc()
  → 堆上分配 _defer{fn: fmt.Println, args: "hello", ...}
  → 插入链表头部

函数返回时：
→ runtime.deferreturn()
  → 遍历链表，逐个执行，逐个释放
```

**开销**：约 **50ns/次**。堆分配 + GC 压力 + 链表操作。

### Go 1.13：栈分配

编译器分析：如果 defer 不在循环里（最多执行一次），就在函数栈帧上直接分配 `_defer` 结构体，省掉堆分配。

```go
type _defer struct {
    // ...
    heap bool  // Go 1.13 新增：标记是堆上还是栈上
    // ...
}
```

`_defer` 结构体里新增了 `heap` 字段来区分分配位置。栈分配不需要 GC 回收，函数返回时栈帧自然销毁。

在 `cmd/go` 的二进制中，370 个 defer 站点有 363 个符合栈分配条件（不在循环中），覆盖率 98%。

**开销**：约 **35ns/次**，比 Go 1.12 快 30%。

### Go 1.14：open-coded defer（内联 defer）

这是最激进的优化。编译器**完全不调用运行时函数**来注册 defer，而是：

1. 用一个 `uint8` 位图（deferBits）记录哪些 defer 已注册
2. 把 defer 函数指针和参数存到栈上的固定 slot
3. 在每个 `return` 语句之前，编译器直接插入代码：按逆序扫描 deferBits，调用对应函数

相当于编译器帮你把 defer 翻译成了 if-else：

```go
// 你写的代码
func foo() {
    defer f1()
    if cond {
        defer f2()
    }
    // ...
    return
}

// 编译器生成的伪代码
func foo() {
    var deferBits uint8
    deferBits |= 1 << 0  // f1 注册
    if cond {
        deferBits |= 1 << 1  // f2 注册
    }
    // ...

    // return 之前：
    if deferBits & (1 << 1) != 0 { f2() }
    if deferBits & (1 << 0) != 0 { f1() }
    return
}
```

**适用条件**（必须全部满足）：

- 没有禁用编译器优化（`-gcflags "-N"` 会关闭）
- defer 数量 ≤ 8（受 uint8 位图限制）
- `defer 数量 × return 语句数量 ≤ 15`（避免生成过多代码）
- defer 不在循环内

不满足条件时会回退到栈分配或堆分配。

**那 panic 时怎么办？** open-coded defer 只在正常返回路径上内联。如果发生 panic，运行时通过编译器编码的元数据（`FUNCDATA_OpenCodedDeferInfo`）找到哪些 defer 已注册但尚未执行，然后执行它们。所以 panic 时 defer 仍然能正确执行。

### 性能对比

| 版本 | 实现方式 | 单次开销 |
|------|---------|---------|
| ≤ Go 1.12 | 堆分配 | ~50ns |
| Go 1.13 | 栈分配 | ~35ns |
| Go 1.14+ | open-coded（内联） | ~6ns |
| 直接函数调用 | 无 defer | ~4.4ns |

从 50ns 到 6ns，open-coded defer 的开销几乎等同于直接函数调用。这就是为什么现在不需要担心 defer 的性能问题。

**一句话总结：Go 1.14 之后，大多数场景下 defer 已经被编译器内联成了普通的函数调用，你唯一需要避免的是在热循环里写 defer（不会走 open-coded 路径，而且每次循环都要注册/执行）。**

---

## defer 的常见陷阱

### 陷阱一：循环里的 defer

```go
func processFiles(paths []string) error {
    for _, path := range paths {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close() // 问题：所有 Close 都要等函数返回才执行
    }
    // 如果 paths 有 10000 个文件，这里会同时打开 10000 个文件描述符
    return nil
}
```

defer 是**函数级别**的，不是块级别的。循环里的 defer 不会在每次迭代结束时执行，而是全部积攒到函数返回时才执行。

**正确做法**：提取成子函数，或者直接手动关闭。

```go
func processFiles(paths []string) error {
    for _, path := range paths {
        if err := processFile(path); err != nil {
            return err
        }
    }
    return nil
}

func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // 每次 processFile 返回时就会 Close
    // ...
    return nil
}
```

### 陷阱二：闭包捕获循环变量

```go
for i := 0; i < 3; i++ {
    defer func() {
        fmt.Println(i) // 捕获的是 i 的地址
    }()
}
// 输出：3 3 3，不是 2 1 0
```

所有闭包引用的是**同一个** `i`，循环结束时 `i` 的值是 3。

**修复**：用参数传值。

```go
for i := 0; i < 3; i++ {
    defer func(n int) {
        fmt.Println(n) // n 是 i 的拷贝
    }(i) // 此刻的 i 被复制给 n
}
// 输出：2 1 0
```

注意：Go 1.22 修改了 for 循环变量的语义（每次迭代创建新变量），这个问题在新版本中已经缓解。但为了代码兼容性和清晰度，参数传值仍然是更好的习惯。

### 陷阱三：defer 里忽略错误

```go
func writeFile(path string, data []byte) error {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer f.Close() // Close 的错误被忽略了！

    _, err = f.Write(data)
    return err
}
```

`f.Close()` 在某些文件系统上可能返回错误（比如 NFS 上的延迟写入失败）。如果你用 defer 关闭并忽略返回值，写入的数据可能实际上没有持久化。

**正确做法**：用命名返回值捕获 Close 的错误。

```go
func writeFile(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer func() {
        closeErr := f.Close()
        if err == nil {
            err = closeErr // 只在没有其他错误时，才把 Close 的错误返回
        }
    }()

    _, err = f.Write(data)
    return err
}
```

---

## panic 的工作原理

### 什么时候会 panic

- **运行时错误**：数组越界、nil 指针解引用、类型断言失败（单值形式）、向已关闭 channel 发送数据
- **主动调用**：`panic("something went wrong")`

### gopanic 做了什么

panic 触发时，运行时调用 `gopanic`（定义在 `runtime/panic.go`），流程：

```
1. 创建 _panic 结构体，挂到 goroutine 的 panic 链上
2. 从当前函数的 defer 链表开始，逐个执行 defer 函数
3. 当前函数的 defer 执行完后，回到调用者，继续执行调用者的 defer
4. 一直回溯到栈底
5. 如果整个过程都没有 recover → 打印 panic 信息和堆栈 → exit(2)
```

关键点：**panic 不是立即崩溃**。它会先把当前 goroutine 所有未执行的 defer 跑一遍。这是 defer 用来做资源清理的基础。

### 嵌套 panic

defer 函数里也可以 panic，这会形成嵌套：

```go
defer func() {
    fmt.Println("defer 1")
    panic("second panic") // defer 执行过程中又 panic 了
}()
panic("first panic")
```

新 panic 会接管旧 panic，继续执行剩余的 defer。如果最终没有 recover，所有嵌套 panic 的信息都会被打印出来。

---

## recover 的工作原理

### 基本用法

```go
func safeCall() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("捕获到 panic:", r)
        }
    }()

    panic("boom")
}

func main() {
    safeCall()
    fmt.Println("程序继续运行") // 能执行到这里
}
```

`recover()` 做了两件事：
1. 返回 panic 的值（如果当前正在 panic 中）
2. 终止 panic 的传播，让函数正常返回

### 为什么 recover 只能在 defer 中直接调用

这不是 Go 的"约定"，是运行时用**栈帧计数**强制检查的。

`gorecover` 函数执行时，会从自身往上回溯栈帧，要求到 `gopanic` 之间**恰好有 1 个非 wrapper 栈帧**（就是 defer 函数本身）。

| 写法 | 中间栈帧数 | 能否 recover |
|------|-----------|-------------|
| `defer func() { recover() }()` | 1（匿名函数） | ✅ |
| `defer recover()` | 0（直接调用） | ❌ |
| `defer func() { doRecover() }()` | 2（匿名函数 + doRecover） | ❌ |
| `defer func() { func() { recover() }() }()` | 2 | ❌ |

第一种是唯一能工作的形式。

**一个常见的错误**：把 recover 封装到工具函数里。

```go
// ❌ 不能工作
func safeRecover() any {
    return recover() // 这里距离 gopanic 隔了两帧
}

defer func() {
    r := safeRecover() // r 永远是 nil
}()
```

封装意味着多了一层栈帧，栈帧数变成 2，recover 直接返回 nil。

### recover 成功后发生什么

recover 把 `_panic` 标记为 `recovered = true`。然后运行时通过 `mcall(recovery)` 跳转回到 defer 的返回点，函数正常返回。后续的调用者完全不知道发生过 panic。

### 跨 goroutine 的 panic 不能 recover

```go
func main() {
    defer func() {
        recover() // 不能捕获子 goroutine 的 panic
    }()

    go func() {
        panic("子 goroutine panic")
    }()

    time.Sleep(time.Second)
}
// 程序崩溃，recover 无效
```

每个 goroutine 有独立的 defer 链和 panic 链，相互之间完全隔离。子 goroutine 的 panic 只能在子 goroutine 内部 recover。

**如果你需要在子 goroutine 中安全地处理 panic：**

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("goroutine panic: %v", r)
        }
    }()

    // 可能 panic 的代码
    riskyOperation()
}()
```

---

## Go 1.21：panic(nil) 的行为变化

Go 1.21 之前有一个问题：

```go
func foo() {
    defer func() {
        r := recover()
        fmt.Println(r) // nil
        // 那到底是没有 panic，还是 panic(nil)？分不清
    }()

    panic(nil)
}
```

`panic(nil)` 后 `recover()` 返回 `nil`，跟"没有 panic"无法区分。

**Go 1.21 的修复**：`panic(nil)` 会被运行时自动替换为 `*runtime.PanicNilError`，这样 `recover()` 一定能返回非 nil 的值。

```go
// Go 1.21+
func foo() {
    defer func() {
        r := recover()
        fmt.Println(r) // &PanicNilError{}，不再是 nil
    }()

    panic(nil)
}
```

如果需要旧行为，可以设置 `GODEBUG=panicnil=1`，或者在 `go.mod` 中声明 `go 1.20` 及更早版本。

---

## 总结

| 知识点 | 记住这一句 |
|--------|-----------|
| 执行顺序 | 后进先出（链表头插 + 头遍历） |
| 参数求值 | defer 声明时立即复制参数值，闭包捕获的是变量地址 |
| 命名返回值 | defer 闭包能修改命名返回值，因为 return 赋值在前、defer 执行在后 |
| 性能演进 | 堆分配 50ns → 栈分配 35ns → open-coded 6ns，现在几乎等于直接调用 |
| open-coded 条件 | defer ≤ 8 个、不在循环里、defer × return ≤ 15 |
| panic 流程 | 创建 _panic → 遍历所有 defer → 没 recover 就打印堆栈退出 |
| recover 限制 | 只能在 defer 函数中直接调用，运行时通过栈帧计数验证 |
| 跨 goroutine | panic 和 recover 不能跨 goroutine，每个 goroutine 独立处理 |
| panic(nil) | Go 1.21+ 自动替换为 PanicNilError，recover 保证返回非 nil |
