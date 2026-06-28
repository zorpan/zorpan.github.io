---
title: Go 核心进阶：错误处理、并发、泛型、反射
date: 2026-06-24 12:00:00
tags:
  - Java对比
  - Go
  - 并发
  - GMP
  - 泛型
  - 反射
  - 面试
categories:
  - Go 进阶之路
---

# 第二阶段：Go 核心进阶

> 深入 Go 语言最核心的并发编程模型与高级特性，掌握 Go 的灵魂所在

---

## 7. 错误处理

### 7.1 error 接口与错误值

```go
// error 是一个极简的接口
type error interface {
    Error() string
}

// 创建错误的几种方式
import "errors"

err1 := errors.New("something went wrong")
err2 := fmt.Errorf("failed to process item %d: %w", id, originalErr) // %w 包装错误

// 自定义错误类型
type NotFoundError struct {
    Resource string
    ID       int64
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s with id %d not found", e.Resource, e.ID)
}
```

### 7.2 错误链与包装（Go 1.13+）

```go
// %w 包装原始错误，保留错误链
func getUser(id int64) (*User, error) {
    user, err := db.Query(id)
    if err != nil {
        return nil, fmt.Errorf("getUser(%d): %w", id, err) // 包装
    }
    return user, nil
}

// errors.Is —— 沿错误链查找目标错误
if errors.Is(err, sql.ErrNoRows) {
    // 处理"未找到"的情况
}

// errors.As —— 沿错误链查找特定类型的错误
var nfErr *NotFoundError
if errors.As(err, &nfErr) {
    fmt.Printf("Resource %s not found\n", nfErr.Resource)
}

// 错误链的展开
// fmt.Errorf("...: %w", err)  → Unwrap() 返回内层错误
// fmt.Errorf("...: %v", err)  → 不产生错误链，Is/As 无法穿透
```

### 7.3 哨兵错误与错误分类

```go
// 哨兵错误：预定义的包级错误值
package sql

var ErrNoRows = errors.New("sql: no rows in result set")

// 使用方通过 errors.Is 判断
if errors.Is(err, sql.ErrNoRows) {
    // 创建新记录
}

// 错误分类的层次设计
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrConflict     = errors.New("conflict")
)

// 包装以添加上下文
func GetUser(id int64) (*User, error) {
    user, err := repo.FindByID(id)
    if errors.Is(err, ErrNotFound) {
        return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
    }
    return user, nil
}
```

### 7.4 panic / recover

```go
// panic 用于不可恢复的错误（程序启动失败、逻辑 bug）
// recover 用于从 panic 中恢复（通常在 defer 中使用）

func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
        }
    }()
    return a / b, nil // b=0 时会 panic
}

// HTTP 中间件中的 panic 捕获
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// 使用原则：
// 1. 不要用 panic 代替 error 返回
// 2. panic 只用于：程序初始化失败、不可能发生的逻辑错误
// 3. 库代码不应该 panic，应该返回 error
// 4. recover 只在 goroutine 顶层或 HTTP 中间件中使用
```

### 7.5 错误处理最佳实践

```go
// 1. 检查错误时尽早返回（Early Return）
func process() error {
    result, err := step1()
    if err != nil {
        return fmt.Errorf("step1: %w", err)
    }

    output, err := step2(result)
    if err != nil {
        return fmt.Errorf("step2: %w", err)
    }

    return step3(output)
}

// 2. 为错误添加上下文（但不要过度包装）
// 好：return fmt.Errorf("saveUser(%d): %w", user.ID, err)
// 坏：return fmt.Errorf("failed: %w", err) // 没有有用信息

// 3. 只在最终处理点检查错误类型，中间层只添加上下文
// 底层
func queryDB(id int) (*User, error) { ... }

// 中间层 —— 只包装，不判断
func getUser(id int) (*User, error) {
    user, err := queryDB(id)
    if err != nil {
        return nil, fmt.Errorf("getUser(%d): %w", id, err)
    }
    return user, nil
}

// 顶层 —— 判断并处理
func handler(w http.ResponseWriter, r *http.Request) {
    user, err := getUser(id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            http.NotFound(w, r)
            return
        }
        http.Error(w, "internal error", 500)
        return
    }
}
```

---

## 8. Goroutine 与并发基础

### 8.1 Goroutine 的本质

```go
// goroutine 是 Go 运行时管理的轻量级协程
// 初始栈仅 2KB（线程默认 1-8MB），可动态增长
// 创建成本极低，可以轻松创建数十万个

go func() {
    fmt.Println("running in goroutine")
}()

// goroutine vs 线程 vs 进程
// 进程：独立地址空间，系统级资源
// 线程：共享进程地址空间，内核调度，切换成本高（~1μs）
// goroutine：用户态调度，切换成本极低（~100ns），栈动态伸缩

// Go runtime 将 M 个 goroutine 映射到 N 个操作系统线程上
// 即 M:N 调度模型
```

### 8.2 GMP 调度模型详解

```go
// G (Goroutine)：代表一个并发任务
// M (Machine)：代表一个操作系统线程
// P (Processor)：代表一个逻辑处理器，持有本地运行队列

// 调度流程：
// 1. 每个 P 有一个本地队列（最大 256 个 G）
// 2. M 从绑定的 P 的本地队列获取 G 来执行
// 3. 本地队列空了，从全局队列偷取（Work Stealing）
// 4. 全局队列也空了，从其他 P 的本地队列偷取一半

// 设置逻辑处理器数量
runtime.GOMAXPROCS(runtime.NumCPU()) // 默认就是 CPU 核心数

// 调度时机：
// - channel 操作阻塞
// - 系统调用阻塞（如文件 IO）
// - time.Sleep
// - 函数调用时的栈检查点
// - GC 时的 STW

// GMP 调度图示：
// ┌──────────────────────────────────────────┐
// │              全局运行队列                   │
// │          [G] [G] [G] [G] [G]              │
// └──────────────────────────────────────────┘
//        ↑ steal          ↑ steal
// ┌─────────────┐  ┌─────────────┐
// │ P0 (本地队列) │  │ P1 (本地队列) │
// │ [G][G][G]    │  │ [G][G][G]    │
// └──────┬──────┘  └──────┬──────┘
//        │                │
//    ┌───┴───┐        ┌───┴───┐
//    │  M0   │        │  M1   │
//    └───────┘        └───────┘
```

### 8.3 Goroutine 泄漏检测与防范

```go
// goroutine 泄漏：goroutine 无法退出，持续占用内存
// 常见原因：
// 1. 向无人接收的 channel 发送
// 2. 从无人发送的 channel 接收
// 3. 死锁
// 4. 无限循环没有退出条件

// 示例：泄漏的 goroutine
func leaky() {
    ch := make(chan int)
    go func() {
        val := <-ch // 永远阻塞，因为没有发送方
        fmt.Println(val)
    }()
    // 函数返回，但 goroutine 永远挂起
}

// 修复：使用 context 控制生命周期
func notLeaky(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-ctx.Done():
            return // 超时或取消时退出
        }
    }()
}

// 检测工具
// go.uber.org/goleak —— 单元测试中检测 goroutine 泄漏
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

// runtime.NumGoroutine() —— 运行时监控
go func() {
    for {
        log.Printf("goroutines: %d\n", runtime.NumGoroutine())
        time.Sleep(10 * time.Second)
    }
}()
```

---

## 9. Channel 与并发模式

### 9.1 Channel 基础与底层原理

```go
// 无缓冲 channel：同步通信（发送方阻塞直到接收方就绪）
ch := make(chan int)

// 有缓冲 channel：异步通信（缓冲满时发送方阻塞）
ch := make(chan int, 10)

// 底层结构 hchan：
// - buf：环形缓冲区
// - sendx/recvx：发送/接收位置
// - sendq/recvq：等待队列（goroutine 链表）
// - lock：互斥锁

// 基本操作
ch <- 42    // 发送
v := <-ch   // 接收
close(ch)   // 关闭 channel

// 关闭规则：
// 1. 只有发送方才应关闭 channel
// 2. 关闭已关闭的 channel 会 panic
// 3. 向已关闭的 channel 发送会 panic
// 4. 从已关闭的 channel 接收会得到零值（不阻塞）
```

### 9.2 Channel 使用模式

```go
// 1. 检测 channel 是否关闭
v, ok := <-ch  // ok=false 表示 channel 已关闭且无数据

// 2. range 遍历 channel（自动在关闭时退出）
for v := range ch {
    fmt.Println(v)
}

// 3. 单向 channel（类型约束）
func producer(ch chan<- int) { // 只能发送
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)
}

func consumer(ch <-chan int) { // 只能接收
    for v := range ch {
        fmt.Println(v)
    }
}

// 4. nil channel 永远阻塞（用于 select 中禁用某分支）
// 5. 已关闭的 channel 永远可读（返回零值）
```

### 9.3 select 多路复用

```go
select {
case msg := <-ch1:
    fmt.Println("from ch1:", msg)
case ch2 <- 42:
    fmt.Println("sent to ch2")
case <-time.After(3 * time.Second):
    fmt.Println("timeout")
case <-ctx.Done():
    fmt.Println("cancelled")
    return
default:
    fmt.Println("no channel ready") // 非阻塞
}

// 超时模式
select {
case result := <-doWork():
    handle(result)
case <-time.After(5 * time.Second):
    fmt.Println("operation timed out")
}

// 非阻塞尝试
select {
case msg := <-ch:
    process(msg)
default:
    // channel 没有数据，跳过
}
```

### 9.4 经典并发模式

```go
// ---- Pipeline（管道模式）----
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// 使用：generate → square → consume
for v := range square(generate(1, 2, 3, 4, 5)) {
    fmt.Println(v) // 1, 4, 9, 16, 25
}

// ---- Fan-Out / Fan-In ----
func fanOut(input <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        channels[i] = square(input) // 多个 worker 并行处理
    }
    return channels
}

func fanIn(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                merged <- v
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(merged)
    }()
    return merged
}

// ---- Worker Pool（工作池）----
func workerPool(jobs <-chan int, results chan<- int, workers int) {
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }(i)
    }
    go func() {
        wg.Wait()
        close(results)
    }()
}

// ---- Or-Done Channel ----
func orDone(done <-chan int, c <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-done:
                return
            case v, ok := <-c:
                if !ok {
                    return
                }
                select {
                case out <- v:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}
```

### 9.5 Channel vs Mutex 选型

```go
// Channel 适用场景：
// - 数据在 goroutine 之间传递（所有权转移）
// - 协调多个 goroutine 的执行顺序
// - 发布/订阅模式
// - 超时控制和取消

// Mutex 适用场景：
// - 保护共享状态（如缓存、计数器）
// - 简单的读写保护
// - 性能敏感的热点路径（channel 有锁开销）

// 经验法则：
// "Don't communicate by sharing memory; share memory by communicating."
// 如果一个数据在同一时刻只有一个 goroutine "拥有"，用 channel
// 如果多个 goroutine 需要读同一个数据，用 RWMutex
```

---

## 10. sync 包与并发原语

### 10.1 Mutex / RWMutex

```go
// Mutex —— 互斥锁
var mu sync.Mutex
var count int

func increment() {
    mu.Lock()
    defer mu.Unlock()
    count++
}

// RWMutex —— 读写锁（读多写少场景性能更好）
var rwmu sync.RWMutex
var cache = make(map[string]string)

func read(key string) string {
    rwmu.RLock()
    defer rwmu.RUnlock()
    return cache[key]
}

func write(key, value string) {
    rwmu.Lock()
    defer rwmu.Unlock()
    cache[key] = value
}

// 注意：Mutex/RWMutex 不能复制（包含 noCopy 标记）
// 如果结构体包含 Mutex，该结构体不能作为值传递，应传指针
```

### 10.2 WaitGroup

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1) // 在启动 goroutine 之前 Add
    go func(id int) {
        defer wg.Done() // 完成时 Done
        doWork(id)
    }(i)
}

wg.Wait() // 阻塞直到所有 Done

// 常见错误：
// 1. 在 goroutine 内部 Add（可能在 Wait 之后才 Add）
// 2. Add 的数量与 Done 不匹配（panic 或永久阻塞）
```

### 10.3 Once

```go
var once sync.Once
var instance *Database

func GetDB() *Database {
    once.Do(func() {
        var err error
        instance, err = connectDB()
        if err != nil {
            panic(err) // Once 内 panic 后，后续调用不再执行 Do
        }
    })
    return instance
}

// 注意：即使传入的函数 panic，Once 也不会再次执行
// 如果需要重试语义，自己实现
```

### 10.4 sync.Map

```go
// 适用于两种场景：
// 1. key 集合稳定，写少读多
// 2. 多个 goroutine 读写不同的 key

var m sync.Map

m.Store("key", "value")
v, ok := m.Load("key")
m.Delete("key")
m.Range(func(key, value any) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true // 返回 false 中止遍历
})

// LoadOrStore：原子性"获取或设置"
v, loaded := m.LoadOrStore("key", "default")

// 不适合的场景：频繁增删 + 需要长度统计 → 用 RWMutex + map
```

### 10.5 sync.Pool

```go
// 对象池：减少频繁的内存分配和 GC 压力
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()      // 重置！
        bufPool.Put(buf) // 归还
    }()
    buf.Write(data)
    // 使用 buf...
}

// 注意：
// Pool 中的对象可能在任意两次 GC 之间被回收
// 不适合做连接池（连接有状态，不应被意外回收）
// 适合临时对象：bytes.Buffer、临时 slice 等
```

### 10.6 context 包

```go
// context 用于在 goroutine 之间传递取消信号、超时、截止时间和请求级值

// 创建方式
ctx := context.Background()             // 空 context，通常用于 main/init
ctx := context.TODO()                   // 不确定用什么 context 时的占位符

// WithCancel —— 手动取消
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // 必须调用，确保资源释放
go func() {
    select {
    case <-ctx.Done():
        fmt.Println("cancelled:", ctx.Err())
        return
    case data := <-workChan:
        process(data)
    }
}()

// WithTimeout —— 超时自动取消
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

// WithDeadline —— 指定截止时间
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(ctx, deadline)
defer cancel()

// WithValue —— 传递请求级元数据（谨慎使用）
ctx = context.WithValue(ctx, "requestID", "abc-123")
id := ctx.Value("requestID")

// Context 传播最佳实践：
// 1. context 作为函数的第一个参数，命名为 ctx
// 2. 不要将 context 存在结构体中
// 3. WithValue 只用于请求级的元数据（trace ID 等），不用于传递业务参数
// 4. 始终调用 cancel 函数（defer cancel()），即使 ctx 已超时
```

---

## 11. 泛型（Go 1.18+）

### 11.1 泛型函数

```go
// 类型参数用方括号 [] 声明
func Map[T any, R any](s []T, f func(T) R) []R {
    result := make([]R, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

// 使用
nums := []int{1, 2, 3, 4}
strs := Map(nums, strconv.Itoa) // []string{"1","2","3","4"}

// Filter
func Filter[T any](s []T, f func(T) bool) []T {
    var result []T
    for _, v := range s {
        if f(v) {
            result = append(result, v)
        }
    }
    return result
}

evens := Filter(nums, func(n int) bool { return n%2 == 0 })
```

### 11.2 类型约束

```go
// 约束是接口，但可以包含类型元素
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~float32 | ~float64
}

// ~ 表示包含底层类型（如 type MyInt int 也满足 ~int）
func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

// 使用标准库约束
import "golang.org/x/exp/constraints"

func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// comparable 约束：可比较的类型（支持 == 和 !=）
func Contains[T comparable](s []T, target T) bool {
    for _, v := range s {
        if v == target {
            return true
        }
    }
    return false
}
```

### 11.3 泛型类型

```go
// 泛型结构体
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

// 泛型接口
type Container[T any] interface {
    Get() T
    Set(T)
}

// 泛型 Map 函数
type Result[T any] struct {
    Value T
    Err   error
}
```

### 11.4 泛型最佳实践

```go
// 1. 不要过度使用泛型——如果接口能解决，就不需要泛型
// 好：用 io.Reader 接口
func Process(r io.Reader) { ... }

// 2. 泛型适合"类型无关的算法"
// 好：排序、过滤、去重等通用操作
func Unique[T comparable](s []T) []T { ... }

// 3. 标准库泛型包
import (
    "slices" // Contains, Sort, Reverse, Index...
    "maps"   // Keys, Values, Clone...
    "cmp"    // Compare, Less...
)

slices.Sort(nums)
slices.Contains(nums, 42)
keys := maps.Keys(myMap)
```

---

## 12. 反射与 unsafe

### 12.1 reflect 基础

```go
// 反射三定律（Rob Pike 提出）：
// 1. 反射可以从接口值获得反射对象
// 2. 反射可以从反射对象获得接口值
// 3. 要修改反射对象，其值必须可设置（settable）

var x float64 = 3.14
v := reflect.ValueOf(x)     // reflect.Value
t := reflect.TypeOf(x)      // reflect.Type
fmt.Println(v.Type())       // float64
fmt.Println(v.Kind())       // float64
fmt.Println(v.Float())      // 3.14
```

### 12.2 结构体反射

```go
type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"email"`
    Age   int    `json:"age" validate:"min=0,max=150"`
}

func inspect(i interface{}) {
    t := reflect.TypeOf(i)
    v := reflect.ValueOf(i)

    if t.Kind() == reflect.Ptr {
        t = t.Elem()
        v = v.Elem()
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        value := v.Field(i)
        jsonTag := field.Tag.Get("json")
        validateTag := field.Tag.Get("validate")
        fmt.Printf("字段: %s, 类型: %s, json: %s, validate: %s, 值: %v\n",
            field.Name, field.Type, jsonTag, validateTag, value)
    }
}
```

### 12.3 通过反射修改值

```go
var x float64 = 3.14

// 错误！v 不可设置
v := reflect.ValueOf(x)
v.SetFloat(2.71) // panic: reflect.Value.SetFloat using unaddressable value

// 正确！传指针
v = reflect.ValueOf(&x).Elem() // Elem() 解引用
v.SetFloat(2.71)
fmt.Println(x) // 2.71

// 修改结构体字段
u := User{Name: "Alice"}
v := reflect.ValueOf(&u).Elem()
v.FieldByName("Name").SetString("Bob")
fmt.Println(u.Name) // "Bob"

// 检查是否可设置
fmt.Println(v.FieldByName("Name").CanSet()) // true
```

### 12.4 unsafe 包

```go
// unsafe.Pointer 可以转换为任意指针类型
// uintptr 可以进行指针运算

// 用途一：结构体内存布局对齐分析
type Example struct {
    a bool    // 1 byte + 7 padding
    b float64 // 8 bytes
    c int32   // 4 bytes + 4 padding
}
// 总大小：24 bytes（不是 13 bytes）

// unsafe.Sizeof / Alignof / Offsetof
fmt.Println(unsafe.Sizeof(Example{}))    // 24
fmt.Println(unsafe.Offsetof(Example{}.b)) // 8

// 用途二：string 和 []byte 零拷贝转换（危险但高效）
func stringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}
func bytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}

// 警告：这些转换绕过了 Go 的安全保障
// 1. 不要修改转换后的 []byte（会破坏 string 的不可变性）
// 2. 使用场景极少，绝大多数情况标准转换足够
// 3. unsafe 的使用会阻止编译器的某些优化
```

---

## 面试专题

### 高频面试题精选

**Q1：GMP 模型是什么？为什么需要 P？**
- G 是 goroutine，M 是系统线程，P 是逻辑处理器
- P 持有本地运行队列和 mcache（内存缓存），减少全局锁竞争
- 没有 P 的话，所有 M 都要竞争全局队列的锁，性能很差
- P 的数量默认等于 CPU 核心数（GOMAXPROCS）

**Q2：向已关闭的 channel 发送/接收会怎样？**
- 向已关闭的 channel 发送 → panic
- 从已关闭的 channel 接收 → 返回零值，ok=false
- 关闭已关闭的 channel → panic
- 从 nil channel 接收/发送 → 永远阻塞

**Q3：context 的 WithCancel、WithTimeout、WithDeadline 有什么区别？**
- WithCancel：手动调用 cancel() 取消
- WithTimeout：设置相对时间（duration），超时自动取消
- WithDeadline：设置绝对时间（time.Time），到达时自动取消
- WithTimeout(ctx, 5s) 等价于 WithDeadline(ctx, time.Now().Add(5s))

**Q4：sync.Once 如果传入的函数 panic 了会怎样？**
- Once 认为函数"已执行"，后续的 Do 调用不会再执行该函数
- 如果需要重试语义，需要自己实现（检查错误后重置标记）

**Q5：泛型和接口的区别？**
- 接口是运行时多态（动态分派），泛型是编译时多态（静态分派）
- 泛型会为不同类型生成具体代码（可能内联优化），性能更好
- 接口适合行为抽象，泛型适合类型无关的算法

---

## 推荐资源

### 书籍
- 《Concurrency in Go》—— Katherine Cox-Buday，并发编程专著
- 《Go 语言高级编程》第七章 —— 深入 goroutine 调度原理
- 《Go 语言设计与实现》—— draveness，深入运行时和编译器

### 在线资源
- [Go 并发模式视频](https://www.youtube.com/watch?v=f6kdp27TYZs) —— Rob Pike 经典演讲
- [GMP 调度器源码分析](https://golang.design/under-the-hood/) —— Go 语言设计与实现
- [Go 泛型教程](https://go.dev/doc/tutorial/generics) —— 官方泛型教程
