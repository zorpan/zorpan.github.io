---
title: Go 语言基础入门
date: 2026-06-20 12:00:00
tags:
  - Java对比
  - Go
  - 基础
  - 语法
  - 面试
categories:
  - Go 进阶之路
---

# 第一阶段：Go 语言基础

> 从零开始掌握 Go 语言的核心语法与基本特性，建立扎实的语言基础

---

## 1. Go 语言入门与环境

### 1.1 Go 语言历史与设计哲学

Go 于 2009 年由 Google 发布，由 Robert Griesemer、Rob Pike 和 Ken Thompson 三位大师设计。它的诞生源于 Google 内部对 C++ 编译速度和大型项目工程化管理的不满。

Go 的三大设计哲学：

- **少即是多（Less is more）**：Go 没有类、继承、泛型（1.18 前）、异常、注解等大量特性，但通过组合、接口、goroutine 等少量核心概念解决绝大多数问题。语法关键字仅 25 个。
- **并发优先（Concurrency first）**：goroutine 和 channel 是语言级的一等公民，不是库的封装。Go 的口号是"不要通过共享内存来通信，而要通过通信来共享内存"。
- **工程化导向（Engineering-oriented）**：强制代码格式化（gofmt）、强制处理错误返回值、统一的项目布局，这些设计让大型团队协作更高效。

### 1.2 开发环境搭建

```bash
# macOS 安装（推荐 Homebrew）
brew install go

# 验证安装
go version
go env GOPATH
go env GOROOT

# 配置国内镜像（加速模块下载）
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GO111MODULE=on
```

**核心环境变量**：

| 变量 | 含义 | 默认值 |
|------|------|--------|
| GOROOT | Go 安装路径 | /usr/local/go |
| GOPATH | 工作空间路径 | ~/go |
| GOPROXY | 模块代理地址 | https://proxy.golang.org,direct |
| GO111MODULE | 模块模式开关 | on（Go 1.16+） |
| GOPRIVATE | 私有模块匹配模式 | 空 |

### 1.3 go 命令工具链

```bash
# 项目初始化
go mod init github.com/yourname/project

# 常用命令速查
go build          # 编译，生成二进制文件
go run            # 编译并运行
go test           # 运行测试
go test -v ./...  # 运行所有包的测试（详细模式）
go test -cover    # 查看测试覆盖率
go vet            # 静态分析，检查可疑代码
go fmt            # 格式化代码
goimports         # 自动管理 import（需单独安装）
go get            # 添加/更新依赖
go mod tidy       # 清理无用依赖、补充缺失依赖
go mod download   # 下载所有依赖到本地缓存
go list -m all    # 查看所有依赖列表
go doc fmt.Println  # 查看文档
```

### 1.4 Go 项目结构与代码组织

```
myproject/
├── cmd/                  # 可执行程序入口
│   └── server/
│       └── main.go
├── internal/             # 私有代码（不可被外部导入）
│   ├── handler/
│   ├── service/
│   └── repository/
├── pkg/                  # 可被外部导入的公共库
│   └── utils/
├── api/                  # API 定义（protobuf、swagger）
├── configs/              # 配置文件
├── scripts/              # 构建/部署脚本
├── go.mod
├── go.sum
├── Makefile
└── README.md
```

> **关键原则**：`internal/` 包只能被其父目录树中的代码导入，这是 Go 编译器强制执行的访问控制。

### 1.5 快速上手：Hello World

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

**要点**：
- 每个 Go 文件必须声明所属的 `package`
- 可执行程序的入口必须是 `package main` 中的 `func main()`
- import 后未使用的包会导致编译错误（Go 的强制规范）

---

## 2. 基本类型与变量

### 2.1 基本数据类型

```go
// 整数类型
var a int = 42          // 平台相关：32 位系统为 int32，64 位为 int64
var b int8 = 127        // -128 ~ 127
var c int16 = 32767
var d int32 = 2147483647
var e int64 = 9223372036854775807
var f uint = 42         // 无符号整数
var g byte = 255        // uint8 的别名，常用于表示原始字节

// 浮点类型
var h float32 = 3.14
var i float64 = 3.141592653589793  // 默认浮点类型，推荐使用

// 布尔与字符串
var j bool = true
var k string = "hello"

// rune 类型
var l rune = '中'  // int32 的别名，表示一个 Unicode 码点
```

**类型选择建议**：
- 整数：不确定大小时用 `int`，需要精确位宽时用 `int32/int64`
- 浮点：始终使用 `float64`，`float32` 精度不够用
- 字节：原始数据用 `byte`，Unicode 字符用 `rune`
- 布尔：Go 中不支持隐式类型转换，`if 1 {}` 会编译报错

### 2.2 变量声明与零值

```go
// 方式一：var 声明（包级别常用）
var name string = "Go"
var age int           // 未赋值时为零值

// 方式二：短声明（函数内部常用，自动推断类型）
score := 99.5
ok := true

// 方式三：批量声明
var (
    host = "localhost"
    port = 8080
    debug = false
)

// 零值机制 —— 每种类型都有明确的零值，不存在"未初始化"
// int/float: 0 / 0.0
// bool:      false
// string:    ""（空字符串）
// pointer/interface/slice/map/channel/func: nil
// struct:    所有字段都是各自类型的零值
```

**面试高频题：new 和 make 的区别**

```go
// new(T) —— 分配零值内存，返回 *T（指针）
p := new(int)   // *int，值为 0
s := new([]int) // *[]int，值为 nil（注意：不是空切片！）

// make(T, args) —— 只用于 slice / map / channel，返回初始化后的 T（非指针）
s2 := make([]int, 0)   // 空切片，已初始化，可直接 append
m  := make(map[string]int) // 空 map，可直接写入
ch := make(chan int, 1)     // 带缓冲的 channel
```

### 2.3 iota 与枚举模式

```go
// iota 是 Go 的常量生成器，在 const 块中从 0 开始递增
type Color int

const (
    Red    Color = iota // 0
    Green               // 1
    Blue                // 2
)

// iota 参与运算
type Size int

const (
    _  = iota             // 跳过 0
    KB Size = 1 << (10 * iota) // 1 << 10 = 1024
    MB                      // 1 << 20 = 1048576
    GB                      // 1 << 30
    TB                      // 1 << 40
)

// 跳值技巧：用 _ 跳过不需要的值
const (
    _   = iota // 0，忽略
    One        // 1
    Two        // 2
    _
    Four       // 4
)
```

### 2.4 类型转换与类型别名

```go
// Go 没有隐式类型转换，所有转换必须显式进行
var i int32 = 100
var j int64 = int64(i)   // 精确转换
var f float64 = float64(i) // 整数转浮点
var k int = int(f)         // 浮点转整数（截断小数）

// 类型别名 vs 类型定义
type MyInt = int   // 类型别名：MyInt 就是 int，两者完全等价
type YourInt int   // 类型定义：YourInt 是全新类型，不能与 int 隐式互转

var a int = 10
var b MyInt = a    // OK，MyInt 就是 int
var c YourInt = a  // 编译错误！YourInt != int
var d YourInt = YourInt(a) // OK，显式转换
```

> **Go 1.9+ 的类型别名**：`type byte = uint8` 和 `type rune = int32` 就是标准库中使用类型别名的例子。

### 2.5 字符串底层原理

```go
// 字符串本质：只读的字节切片
// 底层结构：struct { ptr *byte; len int }
s := "Hello, 世界"

// len 返回字节数，不是字符数
fmt.Println(len(s))            // 13（英文 7 + 中文 6）
fmt.Println(utf8.RuneCountInString(s)) // 9（实际字符数）

// 字符串是不可变的
// s[0] = 'h'  // 编译错误：cannot assign to s[0]

// 遍历方式的区别
for i := 0; i < len(s); i++ {
    fmt.Printf("%x ", s[i]) // 按字节遍历
}
for _, r := range s {
    fmt.Printf("%c ", r)    // 按 rune（Unicode 字符）遍历
}

// 字符串与字节切片的转换（涉及内存拷贝）
bytes := []byte(s)   // string -> []byte
str := string(bytes) // []byte -> string

// 高效拼接：strings.Builder（避免反复拷贝）
var builder strings.Builder
for i := 0; i < 1000; i++ {
    builder.WriteString("hello")
}
result := builder.String()
```

### 2.6 切片（Slice）的本质

```go
// 切片底层结构：struct { ptr *T; len int; cap int }
// 切片是对底层数组的一个"窗口"

// 创建方式
s1 := []int{1, 2, 3}           // 字面量
s2 := make([]int, 5)           // len=5, cap=5
s3 := make([]int, 3, 10)       // len=3, cap=10

// append 与扩容策略
s := make([]int, 0, 2)
s = append(s, 1, 2)     // len=2, cap=2
s = append(s, 3)         // len=3, cap=4（翻倍扩容）
s = append(s, 4, 5, 6)   // len=6, cap=8（继续翻倍）

// Go 1.18+ 扩容策略优化：
// 新容量 < 256 时：翻倍
// 新容量 >= 256 时：newCap += (newCap + 3*256) / 4（约 1.25 倍增长 + 768）

// 切片操作陷阱
a := []int{1, 2, 3, 4, 5}
b := a[1:3]      // [2, 3]，与 a 共享底层数组！
b[0] = 20        // a 变为 [1, 20, 3, 4, 5]！

// 安全复制：使用 copy
c := make([]int, len(b))
copy(c, b)
c[0] = 200       // 不影响 a

// 切片删除元素（不保序）
s = []int{1, 2, 3, 4, 5}
s = append(s[:2], s[3:]...) // 删除索引 2，结果 [1, 2, 4, 5]

// 切片过滤（原地，不分配新内存）
func filter(s []int, f func(int) bool) []int {
    n := 0
    for _, v := range s {
        if f(v) {
            s[n] = v
            n++
        }
    }
    return s[:n]
}
```

### 2.7 Map 的使用与底层原理

```go
// 声明与初始化
m := map[string]int{"Go": 1, "Java": 2}
m2 := make(map[string]int)   // 空 map
m3 := make(map[string]int, 100) // 预分配 hint 容量

// 基本操作
m["Python"] = 3           // 写入
v, ok := m["Go"]          // 读取，ok 表示 key 是否存在
delete(m, "Java")         // 删除

// 遍历顺序是随机的！Go 语言故意为之，防止依赖顺序
for k, v := range m {
    fmt.Printf("%s: %d\n", k, v)
}

// map 的 key 必须是 comparable 类型
// 合法：int, string, pointer, struct（所有字段 comparable）
// 非法：slice, map, func

// map 并发不安全！并发读写会 panic
// 解决方案：sync.RWMutex 或 sync.Map
var mu sync.RWMutex
mu.Lock()
m["key"] = value
mu.Unlock()

mu.RLock()
v := m["key"]
mu.RUnlock()
```

**Map 底层原理简述**：
- 采用哈希表（Hash Table）实现，每个 bucket 存储 8 个键值对
- 哈希冲突使用链地址法（overflow bucket）
- 负载因子超过 6.5 时触发扩容（翻倍或等量扩容）
- 扩容是渐进式的，每次操作迁移少量 bucket，避免一次性卡顿

**面试高频题：map 的有序遍历**

```go
// map 本身无序，需要有序遍历时借助 slice
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys) // 或 slices.Sort(keys)

for _, k := range keys {
    fmt.Printf("%s: %d\n", k, m[k])
}
```

---

## 3. 流程控制

### 3.1 if/else 与条件初始化

```go
// if 可以包含初始化语句
if err := doSomething(); err != nil {
    log.Fatal(err)
}

// err 的作用域仅在 if-else 块内
// 以下写法是常见的错误处理模式
if result, err := compute(); err != nil {
    return err
} else {
    fmt.Println(result) // result 在这里可用
}
```

### 3.2 for 循环（Go 唯一的循环结构）

```go
// 经典三段式
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// while 风格
n := 1
for n < 100 {
    n *= 2
}

// 无限循环
for {
    // break 退出
    break
}

// range 遍历
for i, v := range slice { }    // 索引 + 值
for k, v := range mp { }       // key + 值
for i, r := range str { }      // 字节偏移 + rune
for range slice { }             // 只需要执行次数

// range 的坑：循环变量是地址复用
for _, v := range slice {
    go func() {
        fmt.Println(v) // 可能都是最后一个元素！
    }()
}

// 修复方式（Go 1.22+ 已修复此问题）
for _, v := range slice {
    v := v // 在 Go 1.22 之前需要手动捕获
    go func() {
        fmt.Println(v)
    }()
}
```

> **Go 1.22 重大变更**：range 循环变量改为每次迭代创建新变量，不再需要手动捕获。

### 3.3 switch 与 type switch

```go
// 基本 switch —— 默认不会穿透（不需要 break）
switch day {
case "Monday":
    fmt.Println("周一")
case "Friday":
    fmt.Println("周五")
default:
    fmt.Println("其他")
}

// 多值匹配
switch day {
case "Saturday", "Sunday":
    fmt.Println("周末")
}

// 无条件 switch（替代 if-else 链）
switch {
case score >= 90:
    fmt.Println("优秀")
case score >= 60:
    fmt.Println("及格")
default:
    fmt.Println("不及格")
}

// type switch —— 接口类型判断利器
func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("整数: %d\n", v)
    case string:
        fmt.Printf("字符串: %s\n", v)
    case bool:
        fmt.Printf("布尔: %v\n", v)
    default:
        fmt.Printf("未知类型: %T\n", v)
    }
}
```

### 3.4 defer 语句

```go
// defer 先进后出（LIFO 栈式执行）
func main() {
    defer fmt.Println("第一个 defer")
    defer fmt.Println("第二个 defer")
    defer fmt.Println("第三个 defer")
    fmt.Println("main 函数体")
}
// 输出：main 函数体 → 第三个 → 第二个 → 第一个

// 经典用法：资源释放
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // 确保函数退出时关闭文件
    // ... 处理文件
    return nil
}

// defer 与命名返回值：defer 可以修改命名返回值
func divide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    return a / b, nil
}

// 注意：defer 的参数在声明时就确定了
func main() {
    x := 10
    defer fmt.Println(x) // 打印 10，不是 20
    x = 20
}
```

### 3.5 break/continue 与标签跳转

```go
// 标签用于跳出多层循环
outer:
    for i := 0; i < 10; i++ {
        for j := 0; j < 10; j++ {
            if i+j > 10 {
                break outer // 跳出外层循环
            }
        }
    }

// 标签与 select 结合
loop:
    for {
        select {
        case msg := <-ch:
            if msg == "quit" {
                break loop
            }
            process(msg)
        case <-ctx.Done():
            break loop
        }
    }
```

---

## 4. 函数与闭包

### 4.1 函数定义与多返回值

```go
// 多返回值是 Go 的一等特性
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 3)
if err != nil {
    log.Fatal(err)
}

// 忽略不需要的返回值
result, _ := divide(10, 3)

// 多返回值常用于 (结果, 错误) 模式，这是 Go 的惯例
```

### 4.2 命名返回值与裸 return

```go
// 命名返回值让函数签名更清晰
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return // 裸 return：自动返回 x, y
}

// 注意：裸 return 只应在短函数中使用，否则影响可读性
```

### 4.3 可变参数函数

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// 调用
sum(1, 2, 3)       // 直接传多个参数
s := []int{1, 2, 3}
sum(s...)           // 展开切片

// 可变参数必须是最后一个参数
func logMsg(level string, msgs ...string) {
    for _, msg := range msgs {
        fmt.Printf("[%s] %s\n", level, msg)
    }
}
```

### 4.4 函数作为一等公民

```go
// 函数类型
type HandlerFunc func(http.ResponseWriter, *http.Request)

// 函数作为参数
func process(items []string, fn func(string) bool) []string {
    var result []string
    for _, item := range items {
        if fn(item) {
            result = append(result, item)
        }
    }
    return result
}

// 函数作为返回值
func multiplier(factor int) func(int) int {
    return func(n int) int {
        return n * factor
    }
}

double := multiplier(2)
triple := multiplier(3)
fmt.Println(double(5))  // 10
fmt.Println(triple(5))  // 15
```

### 4.5 闭包的本质与变量捕获

```go
// 闭包 = 函数 + 引用环境
// 闭包捕获的是变量的引用，不是值的拷贝

func counter() func() int {
    count := 0
    return func() int {
        count++       // 捕获外部变量 count
        return count
    }
}

c := counter()
fmt.Println(c()) // 1
fmt.Println(c()) // 2
fmt.Println(c()) // 3

// 陷阱：闭包捕获循环变量
funcs := make([]func(), 5)
for i := 0; i < 5; i++ {
    funcs[i] = func() {
        fmt.Println(i) // 全部打印 5（Go 1.21 及之前）
    }
}
for _, f := range funcs {
    f()
}

// 修复方式一：通过参数传值
for i := 0; i < 5; i++ {
    funcs[i] = func(n int) {
        fmt.Println(n)
    }(i) // 立即传入当前值
}

// 修复方式二：Go 1.22+ 不再需要修复，循环变量每次迭代重新绑定
```

### 4.6 init 函数与包级初始化

```go
// init 函数特点：
// 1. 每个包可以有多个 init 函数（甚至同一文件中多个）
// 2. 不能被手动调用
// 3. 在 main() 之前自动执行
// 4. 执行顺序：依赖包的 init → 当前包的 init → main

// 执行顺序：
// 1. 初始化包级别变量
// 2. 执行 init 函数
// 3. 多个 init 按源文件名字母序执行

// 典型用途
var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("mysql", "dsn...")
    if err != nil {
        log.Fatal(err)
    }
}
```

### 4.7 函数选项模式（Functional Options Pattern）

```go
// 解决"构造函数参数过多"的问题
type Server struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
}

type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func WithTimeout(t time.Duration) Option {
    return func(s *Server) { s.timeout = t }
}

func WithMaxConn(n int) Option {
    return func(s *Server) { s.maxConn = n }
}

func NewServer(host string, opts ...Option) *Server {
    s := &Server{
        host:    host,
        port:    8080,              // 默认值
        timeout: 30 * time.Second,  // 默认值
        maxConn: 100,               // 默认值
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// 使用：清晰且灵活
srv := NewServer("localhost",
    WithPort(9090),
    WithTimeout(60*time.Second),
    WithMaxConn(200),
)
```

---

## 5. 结构体与方法

### 5.1 结构体定义与初始化

```go
type User struct {
    ID       int64
    Name     string
    Email    string
    IsActive bool
}

// 多种初始化方式
u1 := User{ID: 1, Name: "Alice", Email: "alice@example.com", IsActive: true} // 推荐：字段名初始化
u2 := User{1, "Bob", "bob@example.com", false} // 位置初始化（不推荐：字段顺序变化会出错）
u3 := User{Name: "Charlie"} // 部分初始化，其余为零值
u4 := new(User)            // 返回 *User，所有字段为零值
u5 := &User{Name: "Dave"} // 返回 *User（指针），推荐

// 结构体比较：所有字段 comparable 时，结构体也可比较
type Point struct{ X, Y int }
p1 := Point{1, 2}
p2 := Point{1, 2}
fmt.Println(p1 == p2) // true

// 含 slice/map/func 字段的结构体不可比较
```

### 5.2 值接收者 vs 指针接收者

```go
type Counter struct {
    Value int
}

// 值接收者：操作的是副本，不影响原始值
func (c Counter) Get() int {
    return c.Value
}

// 指针接收者：操作原始值
func (c *Counter) Increment() {
    c.Value++
}

// 选择原则：
// 1. 需要修改接收者 → 必须用指针
// 2. 接收者是大结构体 → 用指针（避免拷贝开销）
// 3. 该类型的方法中有一个用了指针 → 所有方法都应用指针（保持一致性）
// 4. 含 sync.Mutex 等不能复制的字段 → 必须用指针
// 5. 不确定时 → 优先用指针

// Go 会自动处理取地址和解引用
var c Counter
c.Increment()   // 编译器自动转为 (&c).Increment()
p := &c
p.Get()          // 编译器自动转为 (*p).Get()
```

### 5.3 结构体嵌入与组合

```go
// Go 用组合代替继承
type Address struct {
    City    string
    Country string
}

func (a Address) Format() string {
    return a.City + ", " + a.Country
}

type Employee struct {
    Name    string
    Address       // 嵌入（匿名字段）
    Salary float64
}

emp := Employee{
    Name:   "Alice",
    Address: Address{City: "Beijing", Country: "China"},
    Salary: 50000,
}

// 字段提升：可以直接访问嵌入字段的字段和方法
fmt.Println(emp.City)       // 等价于 emp.Address.City
fmt.Println(emp.Format())   // 等价于 emp.Address.Format()

// 如果 Employee 自己也定义了 Format()，则会"遮蔽"嵌入的同名方法
```

### 5.4 struct tag

```go
type User struct {
    ID       int64  `json:"id" db:"id" validate:"required"`
    Name     string `json:"name" db:"name" validate:"required,min=2,max=50"`
    Email    string `json:"email,omitempty" db:"email" validate:"required,email"`
    Password string `json:"-" db:"password"` // json:"-" 表示序列化时忽略
}

// 通过反射读取 tag
func printTags(t reflect.Type) {
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("%s: json=%s, db=%s\n",
            field.Name,
            field.Tag.Get("json"),
            field.Tag.Get("db"),
        )
    }
}
printTags(reflect.TypeOf(User{}))
// ID: json=id, db=id
// Name: json=name, db=name
// Email: json=email,omitempty, db=email
// Password: json=-, db=password
```

### 5.5 空结构体的妙用

```go
// struct{} 不占内存空间
var void struct{}

// 用途一：集合（set）
set := make(map[string]struct{})
set["hello"] = struct{}{}
if _, ok := set["hello"]; ok {
    fmt.Println("存在")
}

// 用途二：信号 channel（只传递信号，不传数据）
done := make(chan struct{})
go func() {
    // ... 执行工作
    done <- struct{}{}
}()
<-done

// 用途三：方法只有副作用不需要返回
type Logger struct{}
func (l Logger) Log(msg string) { fmt.Println(msg) }
```

---

## 6. 接口

### 6.1 接口与隐式实现

```go
// 接口定义行为契约
type Writer interface {
    Write(p []byte) (n int, err error)
}

// 隐式实现：不需要显式声明 implements
type FileWriter struct{ path string }

func (fw FileWriter) Write(p []byte) (n int, err error) {
    // 写入文件的实现
    return len(p), nil
}

// FileWriter 自动满足 Writer 接口，零耦合
var w Writer = FileWriter{path: "/tmp/test.txt"}
w.Write([]byte("hello"))

// 接口的价值：定义行为，而不是定义数据
// 小接口 > 大接口（Go 的惯例）
```

### 6.2 接口的底层结构

```go
// 空接口 interface{}（Go 1.18+ 也写作 any）
// 底层结构：eface { _type *_type; data unsafe.Pointer }
// 只存储类型指针和数据指针

// 非空接口（如 io.Reader）
// 底层结构：iface { tab *itab; data unsafe.Pointer }
// itab 包含接口类型、具体类型、方法表

// 理解接口的 nil 陷阱
var p *int = nil
var i interface{} = p
fmt.Println(i == nil) // false！因为 i 的 data 是 nil，但 _type 不是 nil

// 正确的 nil 接口判断：接口为 nil 当且仅当 _type 和 data 都为 nil
```

### 6.3 类型断言与类型选择

```go
var i interface{} = "hello"

// 类型断言
s, ok := i.(string) // s="hello", ok=true
n, ok := i.(int)    // n=0, ok=false（安全写法）

// 不带 ok 的断言：类型不匹配时 panic
// n := i.(int) // panic: interface conversion: interface is string, not int

// 类型选择（推荐方式）
switch v := i.(type) {
case string:
    fmt.Printf("string: %s\n", v)
case int:
    fmt.Printf("int: %d\n", v)
case nil:
    fmt.Println("nil")
default:
    fmt.Printf("unknown: %T\n", v)
}
```

### 6.4 接口设计原则

```go
// Go 的接口哲学：小而精

// 标准库的经典小接口
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}
type Stringer interface {
    String() string
}

// 接口组合
type ReadWriter interface {
    Reader
    Writer
}
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// 设计原则：
// 1. 接口应该由消费者定义（不是由实现者定义）
// 2. 接口应该尽可能小（1-3 个方法）
// 3. 接口应该在使用处定义，不是在实现处
// 4. 不要为了 mock 而接口，而是为了抽象行为而接口

// 反模式：上帝接口
type Service interface {
    CreateUser(ctx context.Context, user *User) error
    GetUser(ctx context.Context, id int64) (*User, error)
    UpdateUser(ctx context.Context, user *User) error
    DeleteUser(ctx context.Context, id int64) error
    ListUsers(ctx context.Context, filter Filter) ([]*User, error)
    // ... 20 个方法
}
```

### 6.5 常用标准库接口

```go
// fmt.Stringer —— 自定义字符串表示
type User struct{ Name string }
func (u User) String() string {
    return fmt.Sprintf("User(%s)", u.Name)
}
fmt.Println(User{"Alice"}) // 输出: User(Alice)

// error 接口
type error interface {
    Error() string
}

// 自定义错误
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s - %s", e.Field, e.Message)
}

// sort.Interface —— 自定义排序
type ByAge []User
func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
sort.Sort(ByAge(users))

// 更简单的方式（Go 1.8+）
sort.Slice(users, func(i, j int) bool {
    return users[i].Age < users[j].Age
})

// Go 1.21+ 最简单
slices.SortFunc(users, func(a, b User) int {
    return cmp.Compare(a.Age, b.Age)
})
```

---

## 面试专题

### 高频面试题精选

**Q1：Go 的 slice 和数组有什么区别？**
- 数组是值类型，长度是类型的一部分（`[3]int` 和 `[5]int` 是不同类型），赋值和传参会拷贝整个数组
- slice 是引用类型，底层是对数组的抽象（指针 + 长度 + 容量），传参仅拷贝 slice header
- 实际开发中很少直接使用数组，几乎都用 slice

**Q2：slice 的扩容机制是怎样的？**
- Go 1.18 之前：容量 < 1024 时翻倍，>= 1024 时增长 1.25 倍
- Go 1.18+：容量 < 256 时翻倍，>= 256 时增长约 1.25 倍 + 768（平滑过渡，减少内存浪费）
- 扩容时会分配新数组并拷贝数据，因此 append 后的切片与原切片不再共享底层数组

**Q3：defer 的执行顺序是什么？**
- LIFO（后进先出）栈式执行
- defer 的参数在声明时就确定值
- defer 可以读取和修改命名返回值
- defer 在 panic 时仍会执行（配合 recover 可以捕获 panic）

**Q4：值接收者和指针接收者的区别？**
- 值接收者操作的是副本，不影响原始值
- 指针接收者直接操作原始值
- 如果类型的任何方法用了指针接收者，那么所有方法都应使用指针接收者（一致性）
- 不可寻址的值（如 map 的 value）只能调用值接收者的方法

**Q5：interface 的底层结构是什么？**
- 空接口 `interface{}`：eface，包含 `_type`（类型信息）和 `data`（数据指针）
- 非空接口：iface，包含 `itab`（接口类型、具体类型、方法表）和 `data`
- 接口为 nil 的条件：type 和 data 都为 nil
- 将一个 nil 的具体类型赋值给接口后，接口 != nil（因为 type 不为空）

**Q6：Go 是面向对象的语言吗？**
- Go 不是传统意义上的面向对象语言，没有类、继承、构造函数
- Go 通过结构体 + 方法 + 接口实现面向对象的核心能力
- Go 用组合（嵌入）代替继承
- Go 的接口是隐式实现的（鸭子类型），解耦更彻底

---

## 推荐资源

### 书籍
- 《Go 程序设计语言》（The Go Programming Language）—— Alan Donovan & Brian Kernighan，Go 圣经
- 《Go 语言高级编程》—— 柴树杉 & 曹春晖，深入底层原理
- 《Go 语言实战》—— William Kennedy，实战导向

### 在线资源
- [Go 官方教程](https://go.dev/tour/) —— 交互式入门教程
- [Go by Example](https://gobyexample.com/) —— 通过示例学习
- [Effective Go](https://go.dev/doc/effective_go) —— 官方最佳实践
- [Go 语言设计哲学](https://go.dev/doc/faq) —— 官方 FAQ

### 练习平台
- [Exercism Go Track](https://exercism.org/tracks/go) —— 循序渐进的练习
- [LeetCode Go 解题](https://leetcode.cn/) —— 刷题时用 Go 实现
