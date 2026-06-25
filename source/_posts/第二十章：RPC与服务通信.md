---
title: RPC 原理与 Dubbo、gRPC 实战
date: 2026-06-18 16:00:00
tags:
  - RPC
  - Dubbo
  - gRPC
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 20.1 RPC 原理

```
RPC 调用流程：
客户端 → 代理（Stub）→ 序列化 → 网络传输 → 反序列化 → 服务端骨架（Skeleton）→ 调用目标方法

核心步骤：
1. 客户端调用代理方法
2. 代理将方法名、参数序列化为字节流
3. 通过网络（TCP/HTTP2）发送到服务端
4. 服务端反序列化，调用实际方法
5. 将结果序列化返回给客户端
6. 客户端反序列化得到结果
```

---

## 20.2 Dubbo

```
架构：
Consumer → Registry（注册中心）→ Provider
   └──────── Monitor ────────┘

核心特性：
- 服务注册与发现（Nacos/Zookeeper）
- 负载均衡（Random/RoundRobin/LeastActive/ConsistentHash）
- 集群容错（Failover/Failfast/Failsafe/Failback/Forking）
- 序列化（Hessian2/Protobuf/Kryo）
- 协议（Dubbo/Triple/HTTP）
```

```java
// 服务提供者
@DubboService
public class UserServiceImpl implements UserService {
    @Override
    public User getUser(Long id) { return userDao.findById(id); }
}

// 服务消费者
@DubboReference
private UserService userService;
```

---

## 20.3 gRPC 与 Protobuf

```protobuf
// user.proto
syntax = "proto3";

service UserService {
    rpc GetUser (GetUserRequest) returns (UserResponse);
    rpc ListUsers (ListUsersRequest) returns (stream UserResponse);  // 流式
}

message GetUserRequest {
    int64 id = 1;
}

message UserResponse {
    int64 id = 1;
    string name = 2;
    int32 age = 3;
}
```

**gRPC 优势：** 基于 HTTP/2、Protobuf 序列化（高效紧凑）、支持流式通信、跨语言。

---

## 20.4 序列化对比

| 序列化 | 速度 | 体积 | 可读性 | 跨语言 |
|--------|------|------|--------|--------|
| JSON | 慢 | 大 | 好 | 好 |
| Protobuf | 快 | 小 | 差 | 好 |
| Hessian | 较快 | 中 | 差 | Java |
| Kryo | 最快 | 小 | 差 | Java |

---

## 面试精选

### 1. RPC 和 HTTP 的区别？

RPC 是一种远程调用范式，可以基于各种协议（TCP/HTTP2）实现。HTTP 是一种传输协议，很多 RPC 框架（如 gRPC）底层使用 HTTP/2 作为传输层。RPC 框架通常比裸 HTTP 调用提供更丰富的服务治理能力（负载均衡、熔断、限流）。

### 2. Dubbo 的核心原理？

基于接口的远程方法调用，通过代理将本地调用转为网络请求。支持多种协议（Dubbo/Triple）、多种注册中心（Nacos/ZK）、多种负载均衡策略和集群容错模式。SPI 机制支持扩展。

### 3. 怎么选择序列化方式？

内部服务调用：Protobuf（高效紧凑）或 Kryo（Java 生态最快）。对外 API：JSON（可读性好，通用性强）。跨语言场景：Protobuf 或 JSON。
