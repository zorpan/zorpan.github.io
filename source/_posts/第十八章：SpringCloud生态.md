---
title: Spring Cloud 微服务治理全解析
date: 2026-06-18 12:00:00
tags:
  - Spring Cloud
  - Nacos
  - Sentinel
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 18.1 服务注册与发现

### 18.1.1 Nacos（推荐）

```yaml
# application.yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev
        group: DEFAULT_GROUP
```

```java
// 服务提供者
@RestController
public class UserController {
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) { return userService.findById(id); }
}

// 服务消费者（OpenFeign）
@FeignClient(name = "user-service", fallbackFactory = UserClientFallback.class)
public interface UserClient {
    @GetMapping("/user/{id}")
    User getUser(@PathVariable Long id);
}
```

**Nacos 核心功能：**
- 服务注册与发现：AP/CP 模式可切换
- 配置管理：动态配置推送
- 命名空间：环境隔离（dev/test/prod）

### 18.1.2 注册中心对比

| 特性 | Nacos | Eureka | Consul |
|------|-------|--------|--------|
| CAP | AP/CP | AP | CP |
| 健康检查 | TCP/HTTP/MySQL | 心跳 | TCP/HTTP/gRPC |
| 配置管理 | 支持 | 不支持 | 支持 |
| 雪崩保护 | 支持 | 支持 | 不支持 |

---

## 18.2 服务调用

### 18.2.1 OpenFeign

```java
@FeignClient(
    name = "order-service",
    url = "${order.service.url}",      // 可选：指定 URL
    fallbackFactory = OrderClientFallback.class  // 降级
)
public interface OrderClient {

    @PostMapping("/order")
    Order createOrder(@RequestBody OrderDTO order);

    @GetMapping("/order/{id}")
    Order getOrder(@PathVariable Long id);
}

// 降级处理
@Component
public class OrderClientFallback implements FallbackFactory<OrderClient> {
    @Override
    public OrderClient create(Throwable cause) {
        return new OrderClient() {
            @Override
            public Order createOrder(OrderDTO order) {
                log.error("创建订单失败", cause);
                throw new BizException("订单服务不可用");
            }
            @Override
            public Order getOrder(Long id) {
                return Order.builder().id(id).status("UNKNOWN").build();
            }
        };
    }
}
```

---

## 18.3 负载均衡

### 18.3.1 Spring Cloud LoadBalancer

```yaml
# 配置负载均衡策略
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false  # 禁用 Ribbon，使用 Spring Cloud LoadBalancer
```

**负载均衡策略：**
- **Round Robin**（轮询，默认）
- **Random**（随机）
- **Weighted**（加权）

---

## 18.4 服务熔断与降级

### 18.4.1 Sentinel

```java
// 资源定义
@SentinelResource(
    value = "getUser",
    blockHandler = "handleBlock",     // 限流/熔断处理
    fallback = "handleFallback"       // 业务异常降级
)
public User getUser(Long id) {
    return userClient.getUser(id);
}

public User handleBlock(Long id, BlockException ex) {
    return User.builder().id(id).name("限流降级").build();
}

public User handleFallback(Long id, Throwable ex) {
    return User.builder().id(id).name("异常降级").build();
}
```

**Sentinel 核心概念：**
- **资源**：被保护的对象（方法、接口）
- **规则**：限流规则、熔断规则、热点规则
- **流控效果**：快速失败、Warm Up（预热）、排队等待

### 18.4.2 熔断策略

| 策略 | 触发条件 | 场景 |
|------|---------|------|
| 慢调用比例 | 响应时间超阈值的比例 | 接口变慢 |
| 异常比例 | 异常请求的比例 | 频繁报错 |
| 异常数 | 异常请求的数量 | 固定阈值 |

---

## 18.5 网关（Spring Cloud Gateway）

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service          # 负载均衡
          predicates:
            - Path=/api/user/**           # 路径匹配
            - Method=GET,POST             # 方法匹配
          filters:
            - StripPrefix=1               # 去掉前缀
            - name: RequestRateLimiter     # 限流
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

**Gateway 核心概念：**
- **Route（路由）**：一组匹配规则和目标 URI
- **Predicate（断言）**：匹配条件（Path、Method、Header 等）
- **Filter（过滤器）**：请求/响应处理（限流、鉴权、日志）

---

## 18.6 配置中心（Nacos Config）

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        group: DEFAULT_GROUP
        namespace: dev
```

```java
// 动态刷新配置
@RefreshScope
@RestController
public class ConfigController {
    @Value("${custom.config.value}")
    private String configValue;
}
```

---

## 18.7 链路追踪

**SkyWalking 架构：**
```
应用（Agent）→ OAP Server（数据收集和分析）→ UI（可视化展示）
```

**核心概念：**
- **Trace**：一次完整请求的链路
- **Span**：链路中的一个操作（如一次 RPC 调用）
- **Segment**：一个服务内的 Span 集合

---

## 18.8 分布式事务（Seata）

### 18.8.1 Seata 三种模式

**AT 模式（推荐）：**
```
1. TM 开启全局事务 → TC 注册全局事务
2. RM 执行本地事务 → TC 注册分支事务
   ├── 一阶段：执行 SQL + 记录 undo_log（before/after image）
   └── 二阶段提交：删除 undo_log
   └── 二阶段回滚：根据 undo_log 反向补偿

优点：对业务无侵入，自动回滚
缺点：需要 undo_log 表，性能有一定影响
```

**TCC 模式：**
```
Try：预留资源（如冻结库存）
Confirm：确认提交（如扣减冻结库存）
Cancel：回滚释放（如解冻库存）

优点：不依赖数据库锁，性能好
缺点：业务侵入大，需要手写三个接口
```

**Saga 模式：**
```
正向操作：T1 → T2 → T3
补偿回滚：C3 → C2 → C1

优点：适合长事务
缺点：补偿逻辑复杂
```

---

## 面试精选

### 1. Nacos 和 Eureka 的区别？

Nacos 支持 AP/CP 切换，Eureka 只支持 AP。Nacos 支持配置管理，Eureka 不支持。Nacos 支持主动健康检查，Eureka 只有心跳。Nacos 有命名空间隔离，社区活跃度更高。

### 2. OpenFeign 的原理？

OpenFeign 基于接口 + 注解定义 HTTP 调用，运行时通过动态代理生成实现类，代理类将方法调用转为 HTTP 请求（通过 LoadBalancer 选择实例）。支持 fallback 降级、请求拦截器、日志级别配置。

### 3. Sentinel 和 Hystrix 的区别？

Sentinel 支持流控（QPS、线程数）、熔断（慢调用、异常比例）、热点限流、系统自适应保护，有控制台可视化配置。Hystrix 只有线程池和信号量隔离。Sentinel 支持 @SentinelResource 注解和 URL 资源自动识别，更易用。

### 4. Spring Cloud Gateway 的工作原理？

请求进入 Gateway → 路由匹配（Predicate）→ 过滤器链处理（Pre Filter → 转发请求 → Post Filter）→ 返回响应。基于 WebFlux 实现非阻塞，性能优于 Zuul 1.x。支持动态路由、限流、重试、熔断。

### 5. Seata AT 模式的原理？

一阶段：执行业务 SQL，记录 undo_log（before image 和 after image），本地事务提交。二阶段提交：删除 undo_log。二阶段回滚：根据 undo_log 生成反向 SQL 执行补偿。对业务代码无侵入，但需要每个数据库建 undo_log 表。

### 6. 怎么选择分布式事务方案？

AT 模式适合大多数场景（对业务无侵入），TCC 模式适合高性能场景（需手写 Try/Confirm/Cancel），Saga 模式适合长事务（如跨多个服务的复杂流程）。大多数场景优先选 AT 模式。
