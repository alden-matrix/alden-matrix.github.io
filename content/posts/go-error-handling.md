---
title: "Go 基础篇：错误处理不只是 if err != nil —— 从设计哲学到工程实践"
date: 2026-03-01
tags: ["Go", "Go基础", "错误处理"]
summary: "Go 的错误处理看起来简单粗暴，但 errors.Is/As、%w/%v 的选择、分层策略这些细节，决定了出问题时你排查的速度。"
draft: false
---

## Go 错误处理的设计哲学

写 Java 或 Python 的人刚接触 Go 时，最不习惯的就是到处写 `if err != nil`。但这其实是 Go 有意为之的设计：**显式优于隐式**。

try-catch 的问题在于：一个大的 try 块里可能有很多行代码会抛异常，出了问题你得逐行排查到底是哪行抛的。而 Go 的方式是每个可能出错的地方都显式检查，出了问题一眼就能定位。

代价是代码看起来啰嗦。但啰嗦的代码和难以排查的 bug 之间，Go 选了前者。

理解这个出发点后，关键就是怎么把错误处理写得既正确又高效。

---

## 三种基本模式

### 1. Sentinel Error：预定义的固定错误值

适合"调用方需要判断是哪种错误"的场景：

```go
// 在包级别定义，变量名以 Err 开头（Go 的命名惯例）
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrDuplicate    = errors.New("duplicate entry")
)
```

标准库里有很多这样的例子：`io.EOF`、`sql.ErrNoRows`、`os.ErrNotExist`。

使用：

```go
func GetUser(id int64) (*User, error) {
    user, err := db.QueryUser(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNotFound // 转换成自己包定义的错误
        }
        return nil, fmt.Errorf("查询用户 %d 失败: %w", id, err)
    }
    return user, nil
}
```

调用方：

```go
user, err := GetUser(123)
if errors.Is(err, ErrNotFound) {
    // 用户不存在，这是预期内的业务场景
    return
}
if err != nil {
    // 其他意外错误
    log.Error("获取用户失败", zap.Error(err))
    return
}
```

**什么时候用**：错误种类是有限的、可枚举的，调用方只需要知道"是不是这种错误"，不需要更多信息。

### 2. 自定义错误类型：带额外信息的错误

当 sentinel error 不够用的时候——比如你需要传递错误码、HTTP 状态码、用户可见的提示信息：

```go
type BizError struct {
    Code    int    // 业务错误码，比如 40001
    Message string // 给用户看的信息
    Cause   error  // 底层错误（可选）
}

func (e *BizError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("[%d] %s: %v", e.Code, e.Message, e.Cause)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

// 实现 Unwrap 方法，让 errors.Is/As 能穿透到 Cause
func (e *BizError) Unwrap() error {
    return e.Cause
}
```

调用方用 `errors.As` 提取具体信息：

```go
var bizErr *BizError
if errors.As(err, &bizErr) {
    // 成功提取到 BizError 类型的错误
    log.Error("业务错误", zap.Int("code", bizErr.Code), zap.Error(bizErr.Cause))
    respondJSON(c, bizErr.Code, bizErr.Message)
    return
}
```

**什么时候用**：调用方不只是想知道"是不是这种错误"，还需要从错误中提取信息（错误码、重试信息等）。

### 3. Error Wrapping：给错误加上下文

Go 1.13 引入的 `%w` 动词，在错误上叠加当前层的上下文信息：

```go
func GetUserProfile(userID int64) (*Profile, error) {
    user, err := userRepo.FindByID(userID)
    if err != nil {
        return nil, fmt.Errorf("获取用户 %d 的档案: %w", userID, err)
    }

    avatar, err := avatarService.Get(user.AvatarID)
    if err != nil {
        return nil, fmt.Errorf("获取头像 %s: %w", user.AvatarID, err)
    }
    // ...
}
```

错误一层层往上抛，最终日志里能看到完整的调用链：

```
处理请求失败: 获取用户 123 的档案: 获取头像 abc: connection refused
```

比单独一个 `connection refused` 有用得多。

---

## errors.Is 和 errors.As 的实现原理

### errors.Is：沿着 wrap 链找值

```go
// 标准库源码简化版
func Is(err, target error) bool {
    if err == nil || target == nil {
        return err == target
    }
    // 关键：先检查 target 是否可比较
    // 如果 target 是不可比较类型（比如包含 slice 的 struct），
    // 直接 == 会 panic，所以要先检查
    isComparable := reflectlite.TypeOf(target).Comparable()
    return is(err, target, isComparable)
}

func is(err error, target error, isComparable bool) bool {
    for {
        if isComparable && err == target {
            return true
        }
        // 如果 err 实现了 Is 方法，调用它（自定义匹配逻辑）
        if x, ok := err.(interface{ Is(error) bool }); ok {
            if x.Is(target) {
                return true
            }
        }
        // Unwrap 继续往下找（Go 1.20+ 支持多错误树）
        switch x := err.(type) {
        case interface{ Unwrap() error }:
            err = x.Unwrap()
            if err == nil { return false }
        case interface{ Unwrap() []error }:
            for _, e := range x.Unwrap() {
                if e != nil && is(e, target, isComparable) {
                    return true
                }
            }
            return false
        default:
            return false
        }
    }
}
```

注意 `isComparable` 检查——这是很多简化版源码分析遗漏的细节。如果 target 是不可比较类型，`==` 会 panic，标准库通过这个检查来避免。

所以 `errors.Is` 的工作方式是：从当前错误开始，沿着 `Unwrap()` 链一层层往下找，直到找到匹配的 target 或者到链尾。

这就是为什么要用 `errors.Is` 而不是 `==`：

```go
original := ErrNotFound
wrapped := fmt.Errorf("查询失败: %w", original)

wrapped == ErrNotFound             // false ← 比较的是两个不同的对象
errors.Is(wrapped, ErrNotFound)    // true  ← 沿着 wrap 链找到了
```

### errors.As：沿着 wrap 链找类型

逻辑类似，但不是比较值，而是检查类型是否可赋值：

```go
// 标准库源码简化版（实际使用 internal/reflectlite，这里用 reflect 表达逻辑）
func as(err error, target any, targetVal Value, targetType Type) bool {
    for {
        // 1. 检查 err 的类型能否赋值给 target 指向的类型
        if TypeOf(err).AssignableTo(targetType) {
            targetVal.Elem().Set(ValueOf(err))
            return true
        }
        // 2. 检查自定义 As 方法
        if x, ok := err.(interface{ As(any) bool }); ok && x.As(target) {
            return true
        }
        // 3. Unwrap 继续往下找（支持单个和多个错误）
        switch x := err.(type) {
        case interface{ Unwrap() error }:
            err = x.Unwrap()
            if err == nil { return false }
        case interface{ Unwrap() []error }:
            for _, e := range x.Unwrap() {
                if e != nil && as(e, target, targetVal, targetType) {
                    return true
                }
            }
            return false
        default:
            return false
        }
    }
}
```

注意：标准库用的是 `internal/reflectlite`（轻量反射包），不是完整的 `reflect`，性能更好。另外 Go 1.20+ 支持 `Unwrap() []error` 分支，会递归遍历所有错误树。

**一句话区分**：Is 看值（"是不是这个错误"），As 看类型（"是不是这类错误"）并提取。

### 自定义 Is 方法

有时候你希望两个不同的错误值被认为是"相等"的：

```go
type HttpError struct {
    StatusCode int
    Message    string
}

func (e *HttpError) Error() string {
    return fmt.Sprintf("HTTP %d: %s", e.StatusCode, e.Message)
}

// 只要状态码相同就认为匹配
func (e *HttpError) Is(target error) bool {
    t, ok := target.(*HttpError)
    if !ok {
        return false
    }
    return e.StatusCode == t.StatusCode
}
```

这样就可以：

```go
err1 := &HttpError{StatusCode: 404, Message: "用户不存在"}
err2 := &HttpError{StatusCode: 404, Message: "商品不存在"}

errors.Is(err1, err2)  // true，因为状态码相同
```

---

## %w vs %v：什么时候切断错误链

```go
// %w：保留错误链，调用方可以用 errors.Is/As 穿透
err1 := fmt.Errorf("操作失败: %w", ErrNotFound)
errors.Is(err1, ErrNotFound) // true

// %v：只拼接错误信息字符串，错误链断了
err2 := fmt.Errorf("操作失败: %v", ErrNotFound)
errors.Is(err2, ErrNotFound) // false
```

大多数时候应该用 `%w`。但有一种场景应该用 `%v`：**你不想暴露底层实现细节给调用方**。

比如你的 service 层用了 GORM，但你不希望 handler 层知道这件事：

```go
// service 层
func (s *UserService) GetUser(id int64) (*User, error) {
    user, err := s.repo.Find(id)
    if err != nil {
        // 用 %v 切断，handler 层不需要知道你用的是 gorm
        return nil, fmt.Errorf("获取用户失败: %v", err)
    }
    return user, nil
}
```

如果用了 `%w`，handler 层就能 `errors.Is(err, gorm.ErrRecordNotFound)`，这意味着你以后换 ORM 就是个破坏性变更。用 `%v` 保持边界清晰。

---

## Go 1.20+：多错误包装

Go 1.20 引入了一个重要特性：一个错误可以同时 wrap 多个错误。

```go
err := fmt.Errorf("多个操作失败: %w, %w", err1, err2)
// 或者
err := errors.Join(err1, err2, err3)
```

`errors.Is` 和 `errors.As` 会遍历所有分支：

```go
err := errors.Join(ErrNotFound, ErrUnauthorized)
errors.Is(err, ErrNotFound)     // true
errors.Is(err, ErrUnauthorized) // true
```

实际应用场景——并发操作收集所有错误：

```go
func ValidateUser(user *User) error {
    var errs []error

    if user.Name == "" {
        errs = append(errs, fmt.Errorf("名称不能为空"))
    }
    if user.Age < 0 {
        errs = append(errs, fmt.Errorf("年龄不能为负数"))
    }
    if user.Email != "" && !isValidEmail(user.Email) {
        errs = append(errs, fmt.Errorf("邮箱格式无效"))
    }

    return errors.Join(errs...) // 如果 errs 为空，返回 nil
}
```

调用方拿到的错误信息会自动换行连接：

```
名称不能为空
年龄不能为负数
邮箱格式无效
```

---

## 工程实践：分层错误处理策略

### 原则一：底层 wrap，顶层打日志

最常见的错误是每一层都打一遍日志，结果同一个错误在日志系统里出现三四次：

```go
// ❌ 错误做法：每层都打日志
func (r *Repo) Query() error {
    err := db.Find(&result).Error
    if err != nil {
        log.Error("repo 查询失败", zap.Error(err))  // 第 1 次
        return err
    }
}
func (s *Service) Do() error {
    err := s.repo.Query()
    if err != nil {
        log.Error("service 处理失败", zap.Error(err))  // 第 2 次
        return err
    }
}
func handler() {
    err := service.Do()
    if err != nil {
        log.Error("handler 失败", zap.Error(err))  // 第 3 次
    }
}
```

正确做法：

```go
// ✅ 底层只 wrap 不打日志，顶层统一打
func (r *Repo) Query() error {
    err := db.Find(&result).Error
    if err != nil {
        return fmt.Errorf("查询用户表: %w", err)  // 只 wrap
    }
}
func (s *Service) Do() error {
    err := s.repo.Query()
    if err != nil {
        return fmt.Errorf("处理业务逻辑: %w", err)  // 只 wrap
    }
}
func handler() {
    err := service.Do()
    if err != nil {
        // 只打一次，但信息完整
        log.Error("请求处理失败", zap.Error(err))
        // 日志内容：请求处理失败: 处理业务逻辑: 查询用户表: connection refused
    }
}
```

### 原则二：在边界层做错误转换

不同层之间不要让错误类型穿透。特别是基础设施层（数据库、缓存、HTTP 客户端）的错误，不应该直接暴露给业务层：

```go
// Service 层：把基础设施错误转成业务错误
func (s *UserService) GetUser(id int64) (*User, error) {
    user, err := s.repo.FindByID(id)
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, ErrUserNotFound  // 转成业务层的 sentinel error
        }
        return nil, fmt.Errorf("获取用户失败: %w", err)
    }
    return user, nil
}

// Handler 层：只需要判断业务错误，不依赖任何 ORM 的类型
func GetUserHandler(c *gin.Context) {
    user, err := userService.GetUser(id)
    if errors.Is(err, ErrUserNotFound) {
        c.JSON(404, gin.H{"error": "用户不存在"})
        return
    }
    if err != nil {
        c.JSON(500, gin.H{"error": "服务内部错误"})
        return
    }
    c.JSON(200, user)
}
```

### 原则三：panic 只用于程序不应该继续运行的场景

```go
// ✅ 适合 panic：程序启动阶段，缺少关键配置
func MustLoadConfig(path string) *Config {
    cfg, err := LoadConfig(path)
    if err != nil {
        panic(fmt.Sprintf("加载配置失败: %v", err))
    }
    return cfg
}

// ❌ 不适合 panic：业务逻辑中的错误
func GetUser(id int64) *User {
    user, err := db.Find(id)
    if err != nil {
        panic(err)  // 不要这样！应该返回 error
    }
    return user
}
```

`Must` 前缀是 Go 社区的惯例，表示"这个函数如果失败就 panic"。标准库里有 `template.Must`、`regexp.MustCompile` 等。只在初始化阶段使用，业务代码里永远返回 error。

---

## 总结

| 场景 | 做法 |
|------|------|
| 调用方需要判断错误种类 | sentinel error + `errors.Is` |
| 需要传递错误码等额外信息 | 自定义错误类型 + `errors.As` |
| 中间层添加上下文 | `fmt.Errorf("xxx: %w", err)` |
| 隐藏底层实现 | `fmt.Errorf("xxx: %v", err)` 切断链 |
| 并发收集多个错误 | `errors.Join`（Go 1.20+） |
| 日志 | 底层只 wrap，顶层统一打 |
| panic | 只用于初始化阶段的致命错误 |

错误处理写得好不好，平时看不出区别，出了线上问题才见真章。一条完整的、带上下文的错误链路，比什么监控工具都管用。
