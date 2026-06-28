---
title: Go 微服务架构
date: 2026-06-20 09:00:00
tags:
  - gRPC
  - 微服务
  - etcd
  - 链路追踪
  - 监控
categories:
  - Go 进阶之路
---

# 第六阶段：微服务架构

> 掌握 Go 微服务生态的核心组件，构建可扩展的分布式系统

---

## 24. 微服务基础

### 24.1 微服务拆分与 Go 的优势

```go
// Go 在微服务领域的天然优势：
// 1. 编译为静态二进制，容器镜像极小（5-20MB），启动毫秒级
// 2. 原生 goroutine 支持高并发
// 3. 标准库网络能力强大（net/http、gRPC 原生支持）
// 4. 内存占用低，适合容器化部署
// 5. Kubernetes、Docker、etcd、Prometheus 均为 Go 编写

// 服务拆分原则：
// - 单一职责：每个服务只负责一个业务域
// - 高内聚低耦合：相关功能聚集，无关功能解耦
// - 数据自治：每个服务拥有自己的数据库
// - 按业务能力拆分：用户服务、订单服务、支付服务
```

### 24.2 服务注册与发现（etcd）

```go
import clientv3 "go.etcd.io/etcd/client/v3"

// 服务注册
cli, _ := clientv3.New(clientv3.Config{
    Endpoints:   []string{"localhost:2379"},
    DialTimeout: 5 * time.Second,
})

// 注册服务（带租约）
lease, _ := cli.Grant(context.Background(), 10) // 10 秒租约

cli.Put(context.Background(),
    "/services/user-service/instance-1",
    `{"host":"192.168.1.10","port":8080}`,
    clientv3.WithLease(lease.ID),
)

// 自动续租
keepAliveCh, _ := cli.KeepAlive(context.Background(), lease.ID)
go func() {
    for range keepAliveCh {
        // 续租成功，继续处理
    }
}()

// 服务发现
resp, _ := cli.Get(context.Background(),
    "/services/user-service/",
    clientv3.WithPrefix(),
)

for _, kv := range resp.Kvs {
    var instance ServiceInstance
    json.Unmarshal(kv.Value, &instance)
    fmt.Printf("发现实例: %s:%d\n", instance.Host, instance.Port)
}

// Watch 变更
watchCh := cli.Watch(context.Background(),
    "/services/user-service/",
    clientv3.WithPrefix(),
)
for watchResp := range watchCh {
    for _, event := range watchResp.Events {
        switch event.Type {
        case clientv3.EventTypePut:
            // 新实例注册或更新
        case clientv3.EventTypeDelete:
            // 实例下线
        }
    }
}
```

### 24.3 配置管理（Viper）

```go
import "github.com/spf13/viper"

// 读取配置文件
viper.SetConfigName("config")     // 文件名
viper.SetConfigType("yaml")       // 文件类型
viper.AddConfigPath("./configs/")  // 搜索路径
viper.AddConfigPath(".")           // 当前目录
viper.ReadInConfig()

// 环境变量覆盖
viper.SetEnvPrefix("APP")
viper.AutomaticEnv() // 自动绑定环境变量

// 读取配置值
port := viper.GetInt("server.port")
dsn := viper.GetString("database.dsn")
debug := viper.GetBool("app.debug")

// 配置结构体绑定
type Config struct {
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
    Redis    RedisConfig    `mapstructure:"redis"`
}

var cfg Config
viper.Unmarshal(&cfg)

// 热更新配置
viper.WatchConfig()
viper.OnConfigChange(func(e fsnotify.Event) {
    log.Println("配置文件变更:", e.Name)
    viper.Unmarshal(&cfg)
})

// 远程配置（支持 etcd、Consul）
viper.AddRemoteProvider("etcd", "localhost:2379", "/config/myapp")
viper.SetConfigType("json")
viper.ReadRemoteConfig()
```

---

## 25. gRPC 进阶

### 25.1 Protobuf 定义与代码生成

```protobuf
// user.proto
syntax = "proto3";
package user;
option go_package = "github.com/example/proto/user";

import "google/protobuf/timestamp.proto";

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
    rpc CreateUser(CreateUserRequest) returns (User);
    rpc WatchUsers(WatchUsersRequest) returns (stream User); // 服务端流
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    google.protobuf.Timestamp created_at = 5;
}

message GetUserRequest {
    int64 id = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
}

message ListUsersResponse {
    repeated User users = 1;
    int64 total = 2;
}
```

```bash
# 代码生成
protoc --go_out=. --go-grpc_out=. user.proto
```

### 25.2 gRPC 服务端实现

```go
import "google.golang.org/grpc"

type userServer struct {
    pb.UnimplementedUserServiceServer
    repo UserRepository
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.repo.FindByID(ctx, req.GetId())
    if err != nil {
        // gRPC 错误处理
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetId())
    }
    return toProto(user), nil
}

// 服务端流式
func (s *userServer) WatchUsers(req *pb.WatchUsersRequest, stream pb.UserService_WatchUsersServer) error {
    ch := s.repo.Subscribe(stream.Context())
    for user := range ch {
        if err := stream.Send(toProto(user)); err != nil {
            return err
        }
    }
    return nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")

    // 拦截器
    server := grpc.NewServer(
        grpc.UnaryInterceptor(loggingInterceptor),
        grpc.StreamInterceptor(streamLoggingInterceptor),
    )

    pb.RegisterUserServiceServer(server, &userServer{repo: repo})

    // 优雅关闭
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
        <-sigCh
        server.GracefulStop()
    }()

    server.Serve(lis)
}
```

### 25.3 gRPC 拦截器

```go
// 一元拦截器（Unary Interceptor）
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()

    // 调用实际处理方法
    resp, err := handler(ctx, req)

    duration := time.Since(start)
    log.Printf("method=%s duration=%v error=%v", info.FullMethod, duration, err)

    return resp, err
}

// 使用 go-grpc-middleware 链式拦截器
import "github.com/grpc-ecosystem/go-grpc-middleware"

server := grpc.NewServer(
    grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
        loggingInterceptor,
        recoveryInterceptor,
        authInterceptor,
        rateLimitInterceptor,
    )),
)
```

### 25.4 gRPC-Gateway

```go
// 将 gRPC 服务暴露为 RESTful API
import (
    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    gw "github.com/example/proto/user" // 生成的 gateway 代码
)

func main() {
    ctx := context.Background()

    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithInsecure()}

    // 注册 gRPC-Gateway 处理器
    gw.RegisterUserServiceHandlerFromEndpoint(ctx, mux, "localhost:50051", opts)

    // HTTP 服务器
    http.ListenAndServe(":8080", mux)
}

// 在 proto 文件中添加 HTTP 映射
// rpc GetUser(GetUserRequest) returns (User) {
//     option (google.api.http) = {
//         get: "/api/v1/users/{id}"
//     };
// }
```

### 25.5 gRPC 错误处理

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

// 返回标准 gRPC 错误
return nil, status.Errorf(codes.NotFound, "user %d not found", id)
return nil, status.Errorf(codes.InvalidArgument, "name is required")
return nil, status.Errorf(codes.PermissionDenied, "insufficient permissions")
return nil, status.Errorf(codes.Unavailable, "service temporarily unavailable")

// 附加错误详情（google.rpc.Status）
st := status.New(codes.InvalidArgument, "invalid request")
detail, _ := st.WithDetails(&errdetails.BadRequest{
    FieldViolations: []*errdetails.BadRequest_FieldViolation{
        {Field: "email", Description: "invalid email format"},
    },
})
return nil, detail.Err()
```

---

## 26. 分布式基础设施

### 26.1 链路追踪（OpenTelemetry）

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/jaeger"
    "go.opentelemetry.io/otel/sdk/trace"
)

// 初始化 TracerProvider
exporter, _ := jaeger.New(jaeger.WithCollectorEndpoint(
    jaeger.WithEndpoint("http://localhost:14268/api/traces"),
))

tp := trace.NewTracerProvider(
    trace.WithBatcher(exporter),
    trace.WithResource(resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName("user-service"),
    )),
)
otel.SetTracerProvider(tp)

// 创建 Span
tracer := otel.Tracer("user-service")
ctx, span := tracer.Start(context.Background(), "GetUser")
defer span.End()

// 记录属性
span.SetAttributes(attribute.String("user.id", userID))
span.RecordError(err) // 记录错误

// 跨服务传播（自动注入/提取 TraceID）
// gRPC 拦截器自动处理
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
```

### 26.2 日志架构（Zap）

```go
import "go.uber.org/zap"

// 结构化日志
logger, _ := zap.NewProduction() // JSON 格式，适合生产
defer logger.Sync()

logger.Info("user created",
    zap.Int64("user_id", 123),
    zap.String("name", "Alice"),
    zap.Duration("latency", time.Since(start)),
)

// 带字段的 Logger
log := logger.With(
    zap.String("service", "user-service"),
    zap.String("version", "v1.0.0"),
)

// 使用 sugared API（更简洁的写法）
sugar := logger.Sugar()
sugar.Infof("user %s created with id %d", name, id)
sugar.Errorw("failed to create user",
    "error", err,
    "user_id", id,
)

// 日志级别：Debug < Info < Warn < Error < DPanic < Panic < Fatal
// 生产环境通常设置 Info 级别

// 日志中注入 TraceID
func WithTraceID(ctx context.Context, logger *zap.Logger) *zap.Logger {
    span := trace.SpanFromContext(ctx)
    if span.SpanContext().HasTraceID() {
        return logger.With(
            zap.String("trace_id", span.SpanContext().TraceID().String()),
            zap.String("span_id", span.SpanContext().SpanID().String()),
        )
    }
    return logger
}
```

### 26.3 Prometheus 监控

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// 定义指标
var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets, // 0.005, 0.01, ..., 10
        },
        []string{"method", "path"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpRequestDuration)
}

// 中间件中记录指标
func MetricsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        duration := time.Since(start).Seconds()

        httpRequestsTotal.WithLabelValues(
            c.Request.Method,
            c.FullPath(),
            strconv.Itoa(c.Writer.Status()),
        ).Inc()

        httpRequestDuration.WithLabelValues(
            c.Request.Method,
            c.FullPath(),
        ).Observe(duration)
    }
}

// 暴露 /metrics 端点
go http.ListenAndServe(":9090", promhttp.Handler())
```

---

## 面试专题

**Q1：gRPC 和 RESTful API 的区别？**
- gRPC 基于 HTTP/2，支持多路复用、头部压缩、双向流
- gRPC 使用 Protobuf 序列化，比 JSON 更小更快
- gRPC 有强类型接口定义（.proto 文件），代码自动生成
- REST 更通用、浏览器可直接访问、调试更方便
- 微服务间通信推荐 gRPC，对外 API 推荐 REST

**Q2：etcd 和 Consul 的区别？**
- etcd：强一致性（Raft），K8s 底层存储，更底层
- Consul：CP+AP 混合模式，内置服务发现、健康检查、KV、Service Mesh
- 选型：K8s 生态用 etcd，独立服务治理用 Consul

**Q3：OpenTelemetry 和 Jaeger 的关系？**
- OpenTelemetry 是统一的可观测性标准（Traces + Metrics + Logs）
- Jaeger 是分布式追踪后端（存储和展示）
- OpenTelemetry 生成数据，Jaeger/Zipkin/Prometheus 存储和展示
- OpenTelemetry 是 CNCF 毕业项目，已成为事实标准

---

## 推荐资源

- [gRPC Go 官方文档](https://grpc.io/docs/languages/go/)
- [OpenTelemetry Go 文档](https://opentelemetry.io/docs/instrumentation/go/)
- [go-zero 微服务框架](https://go-zero.dev/)
- [Kratos 微服务框架](https://go-kratos.dev/)
