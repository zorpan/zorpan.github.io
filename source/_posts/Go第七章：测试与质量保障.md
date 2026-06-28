---
title: Go 测试与质量保障
date: 2026-06-19 12:00:00
tags:
  - 测试
  - 单测
  - Mock
  - 覆盖率
categories:
  - Go 进阶之路
---

# 第七阶段：测试与质量保障

> 掌握 Go 的测试体系与代码质量工具，写出健壮可靠的代码

---

## 27. 单元测试

### 27.1 testing 包基础

```go
// 文件命名：*_test.go
// 函数命名：func TestXxx(t *testing.T)
// 运行命令：go test ./... -v

// calculator.go
func Add(a, b int) int { return a + b }
func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// calculator_test.go
func TestAdd(t *testing.T) {
    result := Add(1, 2)
    if result != 3 {
        t.Errorf("Add(1, 2) = %d, want 3", result)
    }
}
```

### 27.2 表驱动测试（Go 的标志性测试风格）

```go
func TestDivide(t *testing.T) {
    tests := []struct {
        name    string
        a, b    float64
        want    float64
        wantErr bool
    }{
        {"正常除法", 10, 3, 3.3333333333333335, false},
        {"整除", 10, 2, 5, false},
        {"除以零", 10, 0, 0, true},
        {"负数", -10, 2, -5, false},
        {"零除", 0, 5, 0, false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)
            if (err != nil) != tt.wantErr {
                t.Errorf("Divide(%v, %v) error = %v, wantErr %v", tt.a, tt.b, err, tt.wantErr)
                return
            }
            if !tt.wantErr && math.Abs(got-tt.want) > 1e-10 {
                t.Errorf("Divide(%v, %v) = %v, want %v", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### 27.3 子测试与并行测试

```go
func TestUserService(t *testing.T) {
    // t.Run 创建子测试
    t.Run("创建用户", func(t *testing.T) {
        user, err := service.Create("Alice", "alice@example.com")
        if err != nil {
            t.Fatal(err)
        }
        if user.Name != "Alice" {
            t.Errorf("name = %s, want Alice", user.Name)
        }
    })

    t.Run("查询用户", func(t *testing.T) {
        // ...
    })
}

// 并行测试（每个子测试独立运行，不共享状态）
func TestParallel(t *testing.T) {
    tests := []struct{ name, input string }{
        {"小写", "hello"},
        {"大写", "WORLD"},
        {"混合", "GoLang"},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // 标记为并行
            // 注意：并行测试中循环变量需要捕获（Go 1.22 之前）
            result := strings.ToUpper(tt.input)
            t.Log(result)
        })
    }
}
```

### 27.4 TestMain 与测试前置

```go
func TestMain(m *testing.M) {
    // 测试前的全局设置
    db = setupTestDB()
    defer db.Close()

    // 运行所有测试
    code := m.Run()

    // 测试后的清理
    teardownTestDB(db)

    os.Exit(code)
}

// 子测试的 Setup/Teardown
func TestWithSetup(t *testing.T) {
    // Setup
    db := setupTestDB(t)
    defer db.Close() // Teardown

    t.Run("test1", func(t *testing.T) { /* ... */ })
    t.Run("test2", func(t *testing.T) { /* ... */ })
}
```

### 27.5 测试覆盖率

```bash
# 生成覆盖率报告
go test -cover ./...

# 输出覆盖率统计文件
go test -coverprofile=coverage.out ./...

# HTML 可视化
go tool cover -html=coverage.out -o coverage.html

# 查看每个函数的覆盖率
go tool cover -func=coverage.out

# 设置覆盖率阈值（CI 中常用）
go test -coverprofile=coverage.out ./...
coverage=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
if (( $(echo "$coverage < 80" | bc -l) )); then
    echo "Coverage $coverage% is below 80%"
    exit 1
fi
```

### 27.6 基准测试（Benchmark）

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2)
    }
}

func BenchmarkStringConcat(b *testing.B) {
    b.Run("fmt.Sprintf", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = fmt.Sprintf("hello %s %d", "world", 42)
        }
    })
    b.Run("strings.Builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            sb.WriteString("hello ")
            sb.WriteString("world")
            sb.WriteString(" ")
            sb.WriteString(strconv.Itoa(42))
            _ = sb.String()
        }
    })
}

// 运行：go test -bench=. -benchmem -count=5
// -benchmem 显示内存分配统计
// -count=5 多次运行取平均
// -benchtime=10s 运行时间

// 结果解读：
// BenchmarkAdd-8    1000000000    0.25 ns/op    0 B/op    0 allocs/op
//                   ↑ 次数         ↑ 每次耗时    ↑ 每次分配  ↑ 分配次数
```

---

## 28. Mock 与测试策略

### 28.1 接口 Mock

```go
// 定义接口（被测代码依赖的接口）
type UserRepository interface {
    FindByID(ctx context.Context, id int64) (*User, error)
    Save(ctx context.Context, user *User) error
}

// 手工 Mock
type mockUserRepo struct {
    findByIDFn func(ctx context.Context, id int64) (*User, error)
    saveFn     func(ctx context.Context, user *User) error
}

func (m *mockUserRepo) FindByID(ctx context.Context, id int64) (*User, error) {
    return m.findByIDFn(ctx, id)
}

func (m *mockUserRepo) Save(ctx context.Context, user *User) error {
    return m.saveFn(ctx, user)
}

// 使用 Mock 测试
func TestGetUser(t *testing.T) {
    mock := &mockUserRepo{
        findByIDFn: func(ctx context.Context, id int64) (*User, error) {
            return &User{ID: 1, Name: "Alice"}, nil
        },
    }
    svc := NewUserService(mock)

    user, err := svc.GetUser(context.Background(), 1)
    if err != nil {
        t.Fatal(err)
    }
    if user.Name != "Alice" {
        t.Errorf("name = %s, want Alice", user.Name)
    }
}
```

### 28.2 testify 框架

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/stretchr/testify/suite"
)

// assert vs require
// assert：失败后继续执行当前测试
// require：失败后立即终止当前测试

func TestWithTestify(t *testing.T) {
    result := Add(1, 2)

    assert.Equal(t, 3, result)
    assert.NotEmpty(t, result)
    assert.True(t, result > 0)
    require.NoError(t, err) // err 为 nil 才继续
}

// Suite 测试套件
type UserServiceTestSuite struct {
    suite.Suite
    db      *sql.DB
    service *UserService
}

func (s *UserServiceTestSuite) SetupSuite() {
    s.db = setupTestDB()
    s.service = NewUserService(s.db)
}

func (s *UserServiceTestSuite) TearDownSuite() {
    s.db.Close()
}

func (s *UserServiceTestSuite) TestCreateUser() {
    user, err := s.service.Create("Alice", "alice@example.com")
    s.NoError(err)
    s.Equal("Alice", user.Name)
}

func TestUserService(t *testing.T) {
    suite.Run(t, new(UserServiceTestSuite))
}
```

### 28.3 gomock

```bash
# 安装
go install go.uber.org/mock/mockgen@latest

# 生成 mock
mockgen -source=repository.go -destination=mock_repository.go -package=service
```

```go
func TestGetUserWithGomock(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := NewMockUserRepository(ctrl)

    // 设置期望
    mockRepo.EXPECT().
        FindByID(gomock.Any(), int64(1)).
        Return(&User{ID: 1, Name: "Alice"}, nil).
        Times(1) // 期望调用一次

    svc := NewUserService(mockRepo)
    user, err := svc.GetUser(context.Background(), 1)

    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}
```

### 28.4 httptest 测试 HTTP Handler

```go
func TestGetUserHandler(t *testing.T) {
    // 创建请求
    req := httptest.NewRequest("GET", "/users/1", nil)
    req.Header.Set("Authorization", "Bearer test-token")

    // 创建响应记录器
    w := httptest.NewRecorder()

    // 调用 handler
    handler := setupRouter()
    handler.ServeHTTP(w, req)

    // 断言
    assert.Equal(t, 200, w.Code)

    var resp map[string]interface{}
    json.Unmarshal(w.Body.Bytes(), &resp)
    assert.Equal(t, "Alice", resp["name"])
}
```

---

## 29. 代码质量

### 29.1 静态分析工具

```bash
# go vet —— 官方静态分析
go vet ./...

# golangci-lint —— 集成多种 linter（强烈推荐）
# 安装：go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
golangci-lint run ./...

# .golangci.yml 配置
linters:
  enable:
    - errcheck      # 检查未处理的错误
    - gosimple      # 代码简化建议
    - govet         # go vet
    - ineffassign   # 无效赋值
    - staticcheck   # 综合静态分析
    - unused        # 未使用的代码
    - gocritic      # 代码质量检查
    - gofmt         # 格式化
    - misspell      # 拼写检查
    - prealloc      # slice 预分配建议
    - gocyclo       # 圈复杂度
    - bodyclose     # HTTP Body 关闭检查
    - nilerr        # nil 错误检查

linters-settings:
  gocyclo:
    min-complexity: 15
  gocritic:
    enabled-tags:
      - diagnostic
      - performance
      - style
```

### 29.2 Fuzzing 模糊测试（Go 1.18+）

```go
func FuzzReverse(f *testing.F) {
    // 种子语料
    f.Add("hello")
    f.Add("世界")
    f.Add("")

    // Fuzz 目标
    f.Fuzz(func(t *testing.T, s string) {
        rev := Reverse(s)
        doubleRev := Reverse(rev)

        // 不变式：两次反转应该等于原始值
        if s != doubleRev {
            t.Errorf("Reverse(Reverse(%q)) = %q, want %q", s, doubleRev, s)
        }
    })
}

// 运行：go test -fuzz=FuzzReverse -fuzztime=30s
```

### 29.3 依赖注入与可测试性

```go
// 不好的设计：硬编码依赖
type UserService struct{}

func (s *UserService) GetUser(id int64) (*User, error) {
    db, _ := sql.Open("mysql", "dsn...") // 硬编码，无法 mock
    // ...
}

// 好的设计：依赖注入
type UserService struct {
    repo   UserRepository
    cache  Cache
    logger *zap.Logger
}

func NewUserService(repo UserRepository, cache Cache, logger *zap.Logger) *UserService {
    return &UserService{repo: repo, cache: cache, logger: logger}
}

// 测试时注入 mock
func TestGetUser(t *testing.T) {
    svc := NewUserService(mockRepo, mockCache, zap.NewNop())
    // ...
}
```

---

## 面试专题

**Q1：Go 测试中 t.Fatal 和 t.Error 的区别？**
- `t.Error`：记录错误，继续执行当前测试的剩余部分
- `t.Fatal`：记录错误，立即终止当前测试
- `t.Fail`：标记失败，继续执行
- `t.FailNow`：标记失败，立即终止

**Q2：如何测试并发代码？**
- 使用 `t.Parallel()` 并行执行独立测试
- 使用 `race detector`（`go test -race`）检测数据竞争
- 对于 goroutine 泄漏，使用 `goleak.VerifyTestMain`
- 使用 channel 同步测试的开始和结束

**Q3：表驱动测试的优势？**
- 增加测试用例只需添加一行 struct
- 测试逻辑集中，易于维护
- 子测试名称清晰，失败时容易定位
- Go 社区公认的惯用模式

**Q4：如何达到高测试覆盖率？**
- 先测核心业务逻辑
- 用表驱动覆盖边界条件和错误路径
- 接口 Mock 隔离外部依赖
- httptest 测试 HTTP handler
- 不要为了 100% 覆盖率写无意义的测试

---

## 推荐资源

- [Go 测试官方文档](https://go.dev/doc/tutorial/add-a-test)
- [testify 文档](https://pkg.go.dev/github.com/stretchr/testify)
- [gomock 文档](https://github.com/uber-go/mock)
- [golangci-lint 配置参考](https://golangci-lint.run/usage/configuration/)
