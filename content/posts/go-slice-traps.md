---
title: "Go 基础篇：从源码搞懂 slice —— 底层结构、扩容机制与常见陷阱"
date: 2026-03-03
tags: ["Go", "Go基础", "slice", "源码分析"]
summary: "slice 是 Go 里最常用的数据结构，但大多数人只知道'小于 1024 翻倍，大于 1024 增长 25%'。实际上 Go 1.18 已经改了扩容策略，而且最终容量还会被内存对齐修正。这篇文章从 runtime 源码出发，把 slice 讲透。"
draft: false
---

## 从一个 bug 说起

```go
func FilterActive(items []Item) []Item {
    result := items[:0] // 复用底层数组，看起来很聪明
    for _, item := range items {
        if item.Active {
            result = append(result, item)
        }
    }
    return result
}
```

这段代码在大多数情况下能正常工作。但如果调用方在 `FilterActive` 返回后继续使用原始的 `items`，就会发现数据被悄悄改了。

原因是 `result` 和 `items` 共享同一个底层数组，`append` 直接覆盖了原数组的内容。

要理解为什么会这样，得从 slice 的内存结构讲起。

---

## slice 的内存结构

### runtime 中的定义

slice 在 Go runtime 中的定义位于 `src/runtime/slice.go`：

```go
type slice struct {
    array unsafe.Pointer  // 指向底层数组
    len   int             // 当前长度
    cap   int             // 容量
}
```

在 64 位系统上，这三个字段各占 8 字节，总共 **24 字节**。在 32 位系统上是 12 字节。

这意味着 **slice 变量本身不存储数据**。它只是一个 24 字节的描述符（header），指向真正存储数据的底层数组。

### 画个图理解

```
s := []int{10, 20, 30, 40, 50}

    slice header (24 字节)          底层数组（堆上）
    ┌───────────────────┐       ┌────┬────┬────┬────┬────┐
    │ array ────────────┼──────→│ 10 │ 20 │ 30 │ 40 │ 50 │
    │ len = 5           │       └────┴────┴────┴────┴────┘
    │ cap = 5           │         [0]  [1]  [2]  [3]  [4]
    └───────────────────┘
```

当你做切片操作 `a := s[1:3]` 时，并不会复制数据，而是创建一个新的 header 指向同一个底层数组的不同位置：

```
                                     s 的底层数组
    a (新 header)               ┌────┬────┬────┬────┬────┐
    ┌───────────────────┐       │ 10 │ 20 │ 30 │ 40 │ 50 │
    │ array ────────────┼────────────→↑              ↑
    │ len = 2           │          a.array        a 的 cap 到这
    │ cap = 4           │          指向这里
    └───────────────────┘
```

`a` 的 `len` 是 2（只能访问 `[20, 30]`），但 `cap` 是 4（从 `s[1]` 到 `s[4]`，底层数组后面还有空间）。

---

## 共享底层数组：最常见的坑

### 直接修改元素会互相影响

```go
s := []int{10, 20, 30, 40, 50}
a := s[1:3]  // a = [20, 30]

a[0] = 999

fmt.Println(s) // [10 999 30 40 50] ← s[1] 也变了
```

这很好理解，因为 `a[0]` 和 `s[1]` 指向同一块内存。

### append 才是真正危险的

```go
s := []int{10, 20, 30, 40, 50}
a := s[1:3]  // a = [20, 30]，len=2，cap=4

a = append(a, 999)
// a 现在是 [20, 30, 999]
// 但 999 写到了 s[3] 的位置！

fmt.Println(s) // [10 20 30 999 50] ← s[3] 从 40 变成了 999
```

**规则**：append 时如果 `len < cap`，直接在底层数组上写，不会分配新内存。只有当 `len == cap` 时，才会触发扩容，分配新的底层数组。

### 用三参数切片切断共享

Go 支持 `s[low:high:max]` 语法，第三个参数限制 cap：

```go
s := []int{10, 20, 30, 40, 50}
a := s[1:3:3]  // len = 3-1 = 2, cap = 3-1 = 2

a = append(a, 999)
// cap 不够了，Go 分配一个全新的底层数组
// s 完全不受影响

fmt.Println(s) // [10 20 30 40 50] ← 安全
```

**经验**：从一个 slice 切出子 slice 后如果要做 append，用 `s[low:high:high]` 限制 cap，切断与原数组的共享关系。

---

## 扩容机制：不止是翻倍

### 网上说的"小于 1024 翻倍，大于 1024 增长 25%"已经过时了

这是 Go 1.17 及之前的策略，Go 1.18 做了重要调整。旧策略的问题是：当容量从 1023 增长到 1024 时，增长率从 2.0x 突变到 1.25x，存在一个不连续的跳变。

### Go 1.18+ 的新策略：平滑过渡

Go 1.18 在 `growslice` 中引入了新的容量计算逻辑，Go 1.21 将其提取为独立的 `nextslicecap` 函数。算法本身从 1.18 起一直没变，以下是当前版本的代码：

```go
func nextslicecap(newLen, oldCap int) int {
    newcap := oldCap
    doublecap := newcap + newcap
    if newLen > doublecap {
        return newLen
    }

    const threshold = 256
    if oldCap < threshold {
        return doublecap  // 小于 256：直接翻倍
    }
    // 大于等于 256：平滑过渡
    for {
        newcap += (newcap + 3*threshold) >> 2
        if uint(newcap) >= uint(newLen) {
            break
        }
    }

    if newcap <= 0 {
        return newLen // 溢出保护
    }
    return newcap
}
```

### 核心公式拆解

关键在这行：`newcap += (newcap + 3*threshold) >> 2`

展开：

```
newcap = newcap + (newcap + 768) / 4
       = newcap + newcap/4 + 192
       = 1.25 × newcap + 192
```

`192` 这个常数项是设计的精髓：

- **当 oldCap 较小**（刚超过 256）时，192 相对于 `newcap/4` 很大，增长率接近 2.0x
- **当 oldCap 很大**时，192 可以忽略不计，增长率趋近于 1.25x

不同容量下的实际增长率：

| oldCap | 增量 | 实际增长率 |
|--------|------|-----------|
| 256 | (256+768)/4 = 256 | **2.00x** |
| 512 | (512+768)/4 = 320 | **1.63x** |
| 1024 | (1024+768)/4 = 448 | **1.44x** |
| 2048 | (2048+768)/4 = 704 | **1.34x** |
| 4096 | (4096+768)/4 = 1216 | **1.30x** |
| 100000 | (100000+768)/4 = 25192 | **1.25x** |

从 2.0x 到 1.25x 是连续平滑的过渡，没有突变点。

**一句话记住新策略：小于 256 翻倍，大于 256 按 `1.25x + 192` 平滑过渡，越大越接近 1.25 倍。**

### 为什么阈值是 256 而不是 1024

Go 团队的 commit message（`2dda92f`）里说的很清楚：更小的阈值让过渡更加平滑，而选择 256 是因为在追加构建一个超大 slice 时，总的重新分配次数跟旧策略大致相同，不会导致明显的性能退化。

### 但这还没完：内存对齐会修正最终容量

`nextslicecap` 计算出的容量只是"期望值"。Go 的内存分配器按 **size class**（尺寸等级）分配内存，实际分配的可能比请求的多。

`growslice` 中的关键步骤：

```go
// 以 et.Size_ == 1 ([]byte) 为例
capmem = roundupsize(uintptr(newcap))
newcap = int(capmem) // 用实际分配的字节数反算 cap
```

`roundupsize` 会将内存大小向上对齐到最近的 size class。Go runtime 定义了 **68 个 size class**，部分如下：

```
8, 16, 24, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192,
208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576,
640, 704, 768, 896, 1024, 1152, 1280, ...
```

设计原则：**向上取整的浪费率不超过 12.5%**。

举个例子：

```go
s := make([]int64, 0, 5) // int64 = 8 字节，5 * 8 = 40 字节
s = append(s, 1, 2, 3, 4, 5, 6) // 需要 len=6

// nextslicecap(6, 5) → doublecap=10, oldCap < 256 → return 10
// 需要 10 * 8 = 80 字节
// roundupsize(80) → 80 恰好是 size class → 实际分配 80 字节
// newcap = 80 / 8 = 10
```

再看一个不那么"整齐"的例子：

```go
s := make([]int64, 0, 3) // 3 * 8 = 24 字节
s = append(s, 1, 2, 3, 4) // 需要 len=4

// nextslicecap(4, 3) → doublecap=6, oldCap < 256 → return 6
// 需要 6 * 8 = 48 字节
// roundupsize(48) → 48 是 size class → 实际分配 48 字节
// newcap = 48 / 8 = 6
```

但如果元素大小不是 2 的幂：

```go
// 假设某个结构体大小为 13 字节
// nextslicecap 算出 newcap = 10
// 10 * 13 = 130 字节
// roundupsize(130) → 最近的 size class 是 144
// newcap = 144 / 13 = 11（向下取整）
// 最终 cap = 11，比"期望"的 10 多了 1
```

**这就是为什么你打印 cap 时经常发现跟"理论值"不一样**——不是公式错了，是内存对齐修正了最终结果。

### 验证一下

```go
func main() {
    s := make([]int, 0)
    prev := 0
    for i := 0; i < 5000; i++ {
        s = append(s, i)
        if cap(s) != prev {
            growth := float64(cap(s)) / float64(max(prev, 1))
            fmt.Printf("len=%-6d cap=%-6d 增长率=%.2f\n", len(s), cap(s), growth)
            prev = cap(s)
        }
    }
}
```

你会看到增长率从 2.0 逐渐平滑下降到接近 1.25，中间没有任何突变。

---

## growslice 完整流程

把上面的知识串起来，`growslice` 做了这些事：

```
1. 零大小元素快速路径
   - []struct{} 这种不占内存的类型，直接返回 {&zerobase, newLen, newLen}
   - 不分配任何内存

2. 计算期望容量
   - 调用 nextslicecap(newLen, oldCap)

3. 计算实际内存大小并对齐
   - 元素大小 × 期望容量 = 需要的字节数
   - roundupsize 对齐到 size class
   - 用实际字节数反算真正的 cap
   - 针对 et.Size_ == 1、PtrSize、2的幂 有专门的快速路径，避免乘除法

4. 分配内存
   - 不含指针的类型：mallocgc 不需要 GC 扫描
   - 含指针的类型：mallocgc 需要零初始化 + 写屏障

5. 拷贝旧数据到新内存（memmove）

6. 返回新的 slice header
```

---

## nil slice vs 空 slice

```go
var a []int          // nil slice：array=nil, len=0, cap=0
b := []int{}         // 空 slice：array≠nil, len=0, cap=0
c := make([]int, 0)  // 空 slice：array≠nil, len=0, cap=0
```

在大多数操作上行为一致：

```go
len(a) == len(b)           // true, 都是 0
cap(a) == cap(b)           // true, 都是 0
a = append(a, 1)           // 正常
b = append(b, 1)           // 正常
for range a {}             // 正常，零次循环
for range b {}             // 正常，零次循环
```

但 **JSON 序列化** 的结果不同：

```go
jsonA, _ := json.Marshal(a) // "null"
jsonB, _ := json.Marshal(b) // "[]"
```

如果你的 API 返回 `null` 而前端期望 `[]`，就会出问题。很多移动端解析 JSON 时 `null` 和空数组的处理逻辑不同。

**建议**：对外返回的数据，统一用 `make([]T, 0)` 或 `[]T{}` 初始化。

---

## slice 作为函数参数

slice 传参时复制的是 24 字节的 header，不是底层数据。这导致一个反直觉的现象：

```go
func modifyElement(s []int) {
    s[0] = 999  // 会影响调用方 ← 因为底层数组是共享的
}

func appendElement(s []int) {
    s = append(s, 100)  // 不会影响调用方
    // 虽然 append 成功了，但调用方的 header 副本里 len 没变
    // 即使底层数组上已经写了 100，调用方也"看不到"
}
```

**为什么 append 不影响？** 因为 `s` 是 header 的副本。`append` 修改的是函数内部这个副本的 `len`，调用方的 `len` 不受影响。如果触发了扩容，连底层数组都换了，更不会影响调用方。

如果函数需要 append 并让调用方看到，有两种做法：

```go
// 方式一：返回新 slice（推荐）
func appendAndReturn(s []int) []int {
    return append(s, 100)
}

// 方式二：传指针
func appendByPtr(s *[]int) {
    *s = append(*s, 100)
}
```

---

## 从大 slice 截取小部分：注意内存泄漏

```go
data, _ := os.ReadFile("big_file.bin") // 假设 500MB

// 错误做法：
bad := data[:100]
// bad 虽然只有 100 字节，但它引用着 500MB 的底层数组，GC 无法回收

// 正确做法：复制出来
good := make([]byte, 100)
copy(good, data[:100])
// data 没人引用了，GC 可以回收那 500MB
```

子 slice 和父 slice 共享底层数组，只要有一个 slice 还活着，整个底层数组就不会被回收。在处理大数据时要注意这一点。

---

## 预分配的性能差距

如果提前知道 slice 的大小，用 `make([]T, 0, n)` 预分配：

```go
// 不预分配：多次扩容 + 拷贝
func buildBad(n int) []int {
    var s []int
    for i := 0; i < n; i++ {
        s = append(s, i)
    }
    return s
}

// 预分配：1 次分配，0 次扩容
func buildGood(n int) []int {
    s := make([]int, 0, n)
    for i := 0; i < n; i++ {
        s = append(s, i)
    }
    return s
}
```

你可以自己跑 benchmark 验证：

```go
func BenchmarkBuildBad(b *testing.B) {
    for i := 0; i < b.N; i++ { buildBad(10000) }
}
func BenchmarkBuildGood(b *testing.B) {
    for i := 0; i < b.N; i++ { buildGood(10000) }
}
```

在我的机器上（Go 1.24，Apple M2），预分配版本快约 3~4 倍，内存分配从 20 次降到 1 次。具体数字跟机器和 Go 版本有关，但量级差距是稳定的。

---

## 清空 slice 的两种姿势

```go
s := []int{1, 2, 3, 4, 5}

// 方式一：s = s[:0]
// len 归零，cap 不变，底层数组保留
// 适合需要复用的场景（比如在循环里反复填充）
s = s[:0]

// 方式二：s = nil
// 彻底释放，底层数组等 GC 回收
// 适合不再使用的场景
s = nil
```

---

## 总结

| 知识点 | 要记住的 |
|--------|---------|
| 内存结构 | 24 字节的 header（ptr + len + cap），不存储数据 |
| 子切片 | 共享底层数组，修改互相影响 |
| append | cap 够就原地写（危险），不够就扩容分配新数组 |
| 避坑 | `s[low:high:high]` 限制 cap，切断共享 |
| 扩容 | Go 1.18+ 阈值改为 256，增长率从 2.0x 平滑过渡到 1.25x |
| 实际 cap | 被 roundupsize 按 size class 对齐，可能比预期大 |
| nil vs 空 | `json.Marshal` 结果不同：`null` vs `[]` |
| 传参 | 传的是 header 副本，修改元素影响原始，append 不影响 |
| 内存泄漏 | 子 slice 引用大底层数组时，copy 出来让原数组可以被 GC |
| 性能 | 知道大小就 `make([]T, 0, n)`，省掉多次扩容和拷贝 |
