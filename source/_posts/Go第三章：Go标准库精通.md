---
title: Go 标准库精通
date: 2026-06-15 12:00:00
tags:
  - Java对比
  - Go
  - 标准库
  - IO
  - 网络
  - 面试
categories:
  - Go 进阶之路
---

# 第三阶段：Go 标准库精通

> 深入 Go 强大的标准库体系，掌握 IO、网络、编码、并发等核心库的高级用法

---

## 13. IO 与文件操作

### 13.1 io.Reader / io.Writer 接口体系

```go
// Go 的 IO 哲学：用接口抽象一切数据源
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}

// 这两个极简接口统一了：文件、网络、内存缓冲区、压缩流、加密流...
// 任何实现了 Read/Write 的类型都可以无缝组合

// 组合接口
type ReadWriter interface {
    Reader
    Writer
}
type ReadCloser interface {
    Reader
    io.Closer
}

// 实现 Reader 的自定义类型
type StringReader struct {
    data string
    pos  int
}

func (r *StringReader) Read(p []byte) (n int, err error) {
    if r.pos >= len(r.data) {
        return 0, io.EOF
    }
    n = copy(p, r.data[r.pos:])
    r.pos += n
    return n, nil
}
```

### 13.2 io 包核心工具

```go
// io.Copy —— 从 src 读取并写入 dst，直到 EOF 或错误
written, err := io.Copy(dst, src)

// io.CopyN —— 限制拷贝字节数
written, err := io.CopyN(dst, src, 1024) // 最多拷贝 1KB

// io.TeeReader —— 读取时同时写入另一处（类似 tee 命令）
// 用于"一边读取一边计算哈希"
hasher := sha256.New()
reader := io.TeeReader(file, hasher)
data, _ := io.ReadAll(reader)
fmt.Printf("SHA256: %x\n", hasher.Sum(nil))

// io.MultiReader —— 串联多个 Reader
combined := io.MultiReader(header, body, footer)
io.Copy(dst, combined)

// io.LimitReader —— 限制读取字节数
limited := io.LimitReader(file, 1024*1024) // 最多读 1MB

// io.Pipe —— 内存管道（同步读写）
pr, pw := io.Pipe()
go func() {
    pw.Write([]byte("hello"))
    pw.Close()
}()
io.Copy(os.Stdout, pr)
```

### 13.3 bufio 包

```go
// bufio.Reader —— 带缓冲的读取器（减少系统调用次数）
f, _ := os.Open("large.txt")
defer f.Close()
reader := bufio.NewReader(f) // 默认缓冲区 4096 bytes

// 按行读取
line, err := reader.ReadString('\n')
line, isPrefix, err := reader.ReadLine() // 更底层

// bufio.Scanner —— 按分隔符扫描（推荐方式）
f, _ := os.Open("access.log")
defer f.Close()
scanner := bufio.NewScanner(f)
scanner.Split(bufio.ScanLines) // 按行（默认）

for scanner.Scan() {
    line := scanner.Text()
    // 处理每一行
}
if err := scanner.Err(); err != nil {
    log.Fatal(err)
}

// 自定义分隔符
scanner.Split(bufio.ScanWords)      // 按单词
scanner.Split(bufio.ScanRunes)      // 按 rune
scanner.Split(bufio.ScanBytes)      // 按字节

// 增大缓冲区（读取超长行）
scanner.Buffer(make([]byte, 0, 1024*1024), 1024*1024) // 1MB

// bufio.Writer —— 带缓冲的写入器
writer := bufio.NewWriter(f)
fmt.Fprintln(writer, "hello") // 写入缓冲区
writer.Flush()                // 必须 Flush 才会真正写入

// bufio.ReadWriter —— 组合读写
rw := bufio.NewReadWriter(bufio.NewReader(conn), bufio.NewWriter(conn))
```

### 13.4 os 包文件操作

```go
// 读取整个文件（适合小文件）
data, err := os.ReadFile("config.json")

// 写入文件
err := os.WriteFile("output.txt", []byte("hello"), 0644)

// 精细控制
f, err := os.OpenFile("app.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
defer f.Close()

// 常用打开模式
// os.O_RDONLY  —— 只读
// os.O_WRONLY  —— 只写
// os.O_RDWR   —— 读写
// os.O_CREATE  —— 不存在则创建
// os.O_TRUNC   —— 打开时清空
// os.O_APPEND  —— 追加模式

// 文件信息
info, err := os.Stat("file.txt")
info.Name()      // 文件名
info.Size()      // 大小（bytes）
info.Mode()      // 权限
info.ModTime()   // 修改时间
info.IsDir()     // 是否目录

// 目录操作
os.MkdirAll("path/to/dir", 0755)
entries, _ := os.ReadDir(".")
for _, entry := range entries {
    fmt.Println(entry.Name(), entry.IsDir())
}

// filepath.WalkDir —— 递归遍历目录（Go 1.16+ 推荐）
filepath.WalkDir(root, func(path string, d fs.DirEntry, err error) error {
    if err != nil { return err }
    if !d.IsDir() && filepath.Ext(path) == ".go" {
        fmt.Println(path)
    }
    return nil
})
```

---

## 14. 格式化与编码

### 14.1 fmt 包深度使用

```go
// 格式化占位符速查
// %v    值的默认格式
// %+v   结构体会显示字段名
// %#v   Go 语法表示（可直接复制为代码）
// %T    类型
// %d    十进制整数
// %x %X 十六进制
// %f    浮点数
// %.2f  保留两位小数
// %s    字符串
// %q    带引号的字符串
// %p    指针地址
// %b    二进制
// %t    布尔

// 自定义格式化（实现 fmt.Formatter）
type Point struct{ X, Y int }
func (p Point) Format(f fmt.State, verb rune) {
    switch verb {
    case 'v':
        if f.Flag('+') {
            fmt.Fprintf(f, "Point{X:%d, Y:%d}", p.X, p.Y)
        } else {
            fmt.Fprintf(f, "(%d,%d)", p.X, p.Y)
        }
    case 's':
        fmt.Fprintf(f, "(%d,%d)", p.X, p.Y)
    }
}

// Fprint 系列：写入 io.Writer
fmt.Fprintf(w, "count: %d\n", count)

// Sprint 系列：返回字符串
s := fmt.Sprintf("user: %s, age: %d", name, age)

// Scan 系列：从 io.Reader 读取解析
var name string
var age int
fmt.Sscanf("Alice 30", "%s %d", &name, &age)
```

### 14.2 encoding/json

```go
// 序列化与反序列化
type User struct {
    ID       int64  `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email,omitempty"` // 零值时忽略
    Password string `json:"-"`               // 始终忽略
    Age      int    `json:"age,string"`      // 作为字符串序列化
}

u := User{ID: 1, Name: "Alice", Email: ""}

// Marshal
data, err := json.Marshal(u)        // 紧凑格式
data, err := json.MarshalIndent(u, "", "  ") // 缩进格式

// Unmarshal
var u2 User
err := json.Unmarshal(data, &u2)

// 流式编解码（处理大量数据或网络流）
encoder := json.NewEncoder(w) // 写入 io.Writer
encoder.Encode(u)

decoder := json.NewReader(r) // 从 io.Reader 读取
decoder.Decode(&u2)

// 未知结构：map[string]interface{} 或 any
var result map[string]any
json.Unmarshal(data, &result)

// 自定义序列化
func (u User) MarshalJSON() ([]byte, error) {
    type Alias User // 避免递归
    return json.Marshal(&struct {
        DisplayName string `json:"display_name"`
        *Alias
    }{
        DisplayName: u.FirstName + " " + u.LastName,
        Alias:       (*Alias)(&u),
    })
}
```

### 14.3 编码相关包

```go
// encoding/base64
import "encoding/base64"

encoded := base64.StdEncoding.EncodeToString([]byte("hello"))
decoded, _ := base64.StdEncoding.DecodeString(encoded)

// URL 安全的 base64
encoded = base64.URLEncoding.EncodeToString(data)

// encoding/hex
hexStr := hex.EncodeToString([]byte{0xDE, 0xAD, 0xBE, 0xEF})
decoded, _ := hex.DecodeString(hexStr)

// encoding/csv
reader := csv.NewReader(strings.NewReader("a,b,c\n1,2,3"))
records, _ := reader.ReadAll() // [][]string

writer := csv.NewWriter(f)
writer.Write([]string{"name", "age", "city"})
writer.Flush()

// encoding/binary —— 二进制编解码（网络协议、文件格式）
buf := new(bytes.Buffer)
binary.Write(buf, binary.BigEndian, uint32(0x12345678))
var val uint32
binary.Read(buf, binary.BigEndian, &val)
```

---

## 15. 网络编程

### 15.1 net 包基础

```go
// TCP 服务端
listener, err := net.Listen("tcp", ":8080")
if err != nil {
    log.Fatal(err)
}
defer listener.Close()

for {
    conn, err := listener.Accept()
    if err != nil {
        log.Println(err)
        continue
    }
    go handleConn(conn) // 每个连接一个 goroutine
}

func handleConn(conn net.Conn) {
    defer conn.Close()
    scanner := bufio.NewScanner(conn)
    for scanner.Scan() {
        fmt.Fprintf(conn, "Echo: %s\n", scanner.Text())
    }
}

// TCP 客户端
conn, err := net.DialTimeout("tcp", "localhost:8080", 5*time.Second)
defer conn.Close()
fmt.Fprintf(conn, "Hello\n")
response, _ := bufio.NewReader(conn).ReadString('\n')

// UDP
conn, _ := net.ListenPacket("udp", ":9090")
buf := make([]byte, 1024)
n, addr, _ := conn.ReadFrom(buf)
conn.WriteTo([]byte("pong"), addr)
```

### 15.2 net/http 包

```go
// ---- HTTP 服务端 ----
mux := http.NewServeMux()

mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id") // Go 1.22+ 路径参数
    json.NewEncoder(w).Encode(getUser(id))
})

mux.HandleFunc("POST /users", func(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
})

server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  60 * time.Second,
}

// 优雅关闭
ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer cancel()
go func() {
    <-ctx.Done()
    shutdownCtx, _ := context.WithTimeout(context.Background(), 30*time.Second)
    server.Shutdown(shutdownCtx)
}()
server.ListenAndServe()
```

```go
// ---- HTTP 客户端 ----
client := &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}

// GET 请求
resp, err := client.Get("https://api.example.com/users")
defer resp.Body.Close()
body, _ := io.ReadAll(resp.Body)

// POST JSON
data, _ := json.Marshal(user)
resp, err := client.Post(
    "https://api.example.com/users",
    "application/json",
    bytes.NewReader(data),
)

// 带 context 的请求（超时控制）
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
req.Header.Set("Authorization", "Bearer "+token)
resp, err := client.Do(req)
```

### 15.3 HTTP 中间件

```go
type Middleware func(http.Handler) http.Handler

// 日志中间件
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// CORS 中间件
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// 链式组合
func Chain(handler http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

// 使用
handler := Chain(mux, LoggingMiddleware, CORSMiddleware, RecoveryMiddleware)
```

---

## 16. 文本与数据处理

### 16.1 strings 包

```go
import "strings"

// 查找
strings.Contains(s, "hello")         // 是否包含
strings.HasPrefix(s, "http://")      // 前缀
strings.HasSuffix(s, ".go")          // 后缀
strings.Index(s, "world")            // 首次出现位置

// 转换
strings.ToUpper(s) / strings.ToLower(s)
strings.TrimSpace(s)                 // 去除首尾空白
strings.Trim(s, "#")                 // 去除指定字符
strings.ReplaceAll(s, "old", "new")  // 替换所有

// 分割与连接
parts := strings.Split(s, ",")       // 按分隔符分割
parts := strings.Fields(s)           // 按空白分割
joined := strings.Join(parts, "-")   // 连接

// Builder（高效拼接，内部维护 []byte）
var b strings.Builder
b.Grow(1024) // 预分配
b.WriteString("hello")
b.WriteByte(' ')
b.WriteString("world")
result := b.String()
```

### 16.2 time 包

```go
import "time"

// 当前时间
now := time.Now()
now.Unix()        // Unix 时间戳（秒）
now.UnixMilli()   // 毫秒
now.UnixNano()    // 纳秒

// 解析与格式化（Go 用参考时间，不是 Y-m-d）
// 参考时间：Mon Jan 2 15:04:05 MST 2006
// 简记：1月2日下午3点4分5秒2006年MST-0700
t, _ := time.Parse("2006-01-02", "2024-01-15")
formatted := now.Format("2006-01-02 15:04:05")

// 常用格式常量
time.RFC3339     // "2006-01-02T15:04:05Z07:00"
time.DateTime    // "2006-01-02 15:04:05"（Go 1.20+）
time.DateOnly    // "2006-01-02"（Go 1.20+）

// 时间运算
tomorrow := now.Add(24 * time.Hour)
duration := tomorrow.Sub(now) // time.Duration
duration.Seconds()
duration.Minutes()
duration.Hours()

// 定时器
timer := time.NewTimer(5 * time.Second)
<-timer.C // 阻塞到超时

ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()
for t := range ticker.C {
    fmt.Println(t) // 每秒执行
}

// 超时控制
select {
case <-time.After(3 * time.Second):
    fmt.Println("timeout")
case result := <-ch:
    fmt.Println("got result:", result)
}
```

### 16.3 sort 与 slices 包

```go
// Go 1.21+ 推荐使用 slices 包
import "slices"

nums := []int{3, 1, 4, 1, 5, 9}
slices.Sort(nums)              // 升序
slices.SortFunc(users, func(a, b User) int {
    return cmp.Compare(a.Age, b.Age) // 自定义排序
})
slices.Reverse(nums)           // 反转
slices.Contains(nums, 4)      // 是否包含
slices.Index(nums, 4)         // 查找索引
slices.Compact(nums)           // 去重（需先排序）
slices.Min(nums) / slices.Max(nums)

// 旧版本 sort 包（仍可用）
sort.Ints(nums)
sort.Strings(strs)
sort.Slice(users, func(i, j int) bool {
    return users[i].Age < users[j].Age
})

// 自定义排序接口
type ByAge []User
func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
sort.Sort(ByAge(users))
```

---

## 17. 并发工具标准库

### 17.1 runtime 包

```go
import "runtime"

// goroutine 信息
runtime.NumGoroutine()  // 当前 goroutine 数量
runtime.NumCPU()        // CPU 核心数
runtime.GOMAXPROCS(0)   // 获取当前 P 的数量

// 让出执行权
runtime.Gosched()

// 触发 GC
runtime.GC()

// 内存统计
var m runtime.MemStats
runtime.ReadMemStats(&m)
fmt.Printf("Alloc: %d MB\n", m.Alloc/1024/1024)
fmt.Printf("TotalAlloc: %d MB\n", m.TotalAlloc/1024/1024)
fmt.Printf("NumGC: %d\n", m.NumGC)

// 栈信息（调试用）
buf := make([]byte, 1024)
n := runtime.Stack(buf, false) // 当前 goroutine
n = runtime.Stack(buf, true)   // 所有 goroutine

// Goexit：终止当前 goroutine（会执行所有 defer）
runtime.Goexit()

// Caller / Callers —— 获取调用栈信息
pc, file, line, ok := runtime.Caller(1)
```

### 17.2 pprof 性能分析

```go
// 方式一：HTTP 端点（最常用）
import _ "net/http/pprof"

go func() {
    http.ListenAndServe("localhost:6060", nil)
}()

// 访问方式：
// go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30  (CPU)
// go tool pprof http://localhost:6060/debug/pprof/heap               (内存)
// go tool pprof http://localhost:6060/debug/pprof/goroutine           (goroutine)
// go tool pprof http://localhost:6060/debug/pprof/block               (阻塞)

// 方式二：代码中直接使用
import "runtime/pprof"

// CPU profile
f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()

// 内存 profile
f, _ := os.Create("mem.prof")
pprof.WriteHeapProfile(f)

// 方式三：go tool trace（更底层的调度追踪）
f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()
// go tool trace trace.out
```

---

## 面试专题

**Q1：io.Reader 和 io.Writer 的 Read/Write 方法返回值含义？**
- Read：返回读取的字节数 n 和错误 err。n 可能 > 0 即使 err != EOF
- 当读到数据末尾时，最后一次 Read 返回 0, io.EOF
- Write：返回写入的字节数 n。如果 n < len(p)，则 err != nil

**Q2：bufio.Scanner 和 bufio.Reader 的区别？**
- Scanner 提供更方便的按行/按词扫描接口，适合文本处理
- Reader 提供更底层的 Read 方法，适合二进制或自定义协议
- Scanner 有最大 token 大小限制，Reader 没有

**Q3：time.Parse 和 time.ParseInLocation 的区别？**
- Parse 使用字符串中指定的时区（如 "+08:00"）
- ParseInLocation 使用指定的 Location 解析没有时区信息的时间字符串
- 服务器端通常使用 UTC 存储，展示时再转为用户时区

**Q4：为什么 Go 用 "2006-01-02 15:04:05" 作为格式参考时间？**
- 这是 Go 的独特设计，用一个具体的参考时间点来定义格式
- 记忆口诀：月-日-时-分-秒-年-时区 → 1-2-3-4-5-6-7（MST=-0700）

---

## 推荐资源

- [Go 标准库文档](https://pkg.go.dev/std) —— 官方包文档
- [Go 标准库源码阅读](https://github.com/golang/go/tree/master/src) —— 直接读源码
- [Go 语言标准库视频教程](https://www.bilibili.com/video/BV1gf4y1r7CV/) —— 中文讲解
