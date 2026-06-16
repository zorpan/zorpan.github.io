---
title: Java 开发工程师 · 深入浅出学习进阶大纲
date: 2026-06-15 10:00:00
tags:
  - Java
  - 学习路线
  - 进阶
categories:
  - Java 进阶之路
---

> 从入门到架构师的完整学习路线，涵盖 Java 全栈核心知识体系。本文是系列文章的总纲，后续将逐章展开深入讲解。

<!-- more -->

## 第一阶段：Java 基础夯实

### 1. Java 语言基础
- 1.1 数据类型与变量（基本类型、包装类、自动装箱拆箱）
- 1.2 运算符与流程控制
- 1.3 数组与字符串（String、StringBuilder、StringBuffer）
- 1.4 方法定义与递归
- 1.5 面向对象基础（封装、继承、多态）
- 1.6 抽象类与接口的设计原则
- 1.7 内部类与匿名类
- 1.8 枚举与注解（Annotation）

### 2. 异常处理
- 2.1 异常体系结构（Error vs Exception）
- 2.2 受检异常与非受检异常
- 2.3 try-with-resources 与资源管理
- 2.4 自定义异常设计规范
- 2.5 异常处理最佳实践

### 3. 集合框架
- 3.1 Collection 体系全景图
- 3.2 List：ArrayList、LinkedList、Vector 对比
- 3.3 Set：HashSet、TreeSet、LinkedHashSet
- 3.4 Map：HashMap、TreeMap、LinkedHashMap、Hashtable
- 3.5 Queue 与 Deque
- 3.6 Collections 工具类
- 3.7 集合选型原则与性能对比

### 4. 泛型编程
- 4.1 泛型类与泛型接口
- 4.2 泛型方法与类型推断
- 4.3 通配符（? extends / ? super）与 PECS 原则
- 4.4 类型擦除的本质与限制
- 4.5 泛型与反射的协作

### 5. I/O 与 NIO
- 5.1 字节流与字符流
- 5.2 缓冲流与装饰器模式
- 5.3 文件操作与序列化
- 5.4 NIO 核心：Buffer、Channel、Selector
- 5.5 NIO.2 与 Path/Files API
- 5.6 文件监控服务（WatchService）

### 6. Java 8+ 新特性
- 6.1 Lambda 表达式与函数式接口
- 6.2 Stream API（中间操作、终止操作、并行流）
- 6.3 Optional 优雅处理空值
- 6.4 方法引用与构造器引用
- 6.5 接口默认方法与静态方法
- 6.6 Java 9~21 重要新特性概览（var、Record、Sealed Classes、Pattern Matching、Virtual Threads）

---

## 第二阶段：Java 核心进阶

### 7. 面向对象深入设计
- 7.1 设计模式（23 种经典模式详解）
  - 创建型：单例、工厂方法、抽象工厂、建造者、原型
  - 结构型：适配器、桥接、组合、装饰器、外观、享元、代理
  - 行为型：责任链、命令、迭代器、观察者、策略、模板方法、状态、中介者
- 7.2 SOLID 原则与实战应用
- 7.3 DRY、KISS、YAGNI 编程原则
- 7.4 领域驱动设计（DDD）入门

### 8. 反射与动态代理
- 8.1 Class 类与类加载过程
- 8.2 反射 API（Constructor、Method、Field）
- 8.3 反射性能分析与优化
- 8.4 JDK 动态代理与 Proxy 类
- 8.5 CGLIB 动态代理
- 8.6 字节码操作：ASM、Javassist、Byte Buddy

### 9. 多线程与并发编程
- 9.1 线程基础（创建、生命周期、线程方法）
- 9.2 synchronized 与锁升级（偏向锁→轻量级锁→重量级锁）
- 9.3 volatile 与内存可见性
- 9.4 JMM（Java 内存模型）与 happens-before
- 9.5 Lock 体系（ReentrantLock、ReadWriteLock、StampedLock）
- 9.6 AQS（AbstractQueuedSynchronizer）原理
- 9.7 线程池（ThreadPoolExecutor 参数、执行流程、拒绝策略）
- 9.8 并发集合（ConcurrentHashMap、CopyOnWriteArrayList）
- 9.9 原子类与 CAS 操作
- 9.10 CountDownLatch、CyclicBarrier、Semaphore
- 9.11 CompletableFuture 异步编程
- 9.12 ForkJoin 框架与工作窃取
- 9.13 ThreadLocal 原理与内存泄漏
- 9.14 死锁分析与排查

### 10. JVM 深度解析
- 10.1 JVM 内存模型（堆、栈、方法区、程序计数器）
- 10.2 垃圾回收算法（标记-清除、标记-整理、复制、分代）
- 10.3 垃圾收集器（Serial、ParNew、Parallel、CMS、G1、ZGC、Shenandoah）
- 10.4 类加载机制与双亲委派模型
- 10.5 自定义类加载器
- 10.6 JIT 编译与逃逸分析
- 10.7 JVM 调优实战（JPS、Jstack、Jmap、Jstat、VisualVM、Arthas）
- 10.8 OOM 问题排查与线上案例
- 10.9 字节码结构与执行引擎

---

## 第三阶段：数据库与持久层

### 11. MySQL 数据库
- 11.1 SQL 语法进阶与执行顺序
- 11.2 索引原理（B+ 树、Hash 索引、全文索引）
- 11.3 索引优化策略（覆盖索引、最左前缀、索引下推）
- 11.4 EXPLAIN 执行计划分析
- 11.5 事务与隔离级别（ACID、脏读、幻读、MVCC）
- 11.6 InnoDB 存储引擎架构（Buffer Pool、Redo Log、Undo Log）
- 11.7 锁机制（行锁、表锁、间隙锁、临键锁）
- 11.8 分库分表策略（垂直拆分、水平拆分、ShardingSphere）
- 11.9 读写分离与主从复制
- 11.10 慢 SQL 分析与优化

### 12. Redis
- 12.1 数据类型及底层数据结构（SDS、ziplist、quicklist、skiplist、dict）
- 12.2 持久化机制（RDB、AOF、混合持久化）
- 12.3 内存淘汰策略
- 12.4 Redis 单线程模型与多线程演进
- 12.5 主从复制、哨兵机制、Cluster 集群
- 12.6 分布式锁（SETNX、Redisson、RedLock）
- 12.7 缓存穿透、击穿、雪崩解决方案
- 12.8 延迟队列与 Stream
- 12.9 Lua 脚本与事务

---

## 第四阶段：主流框架源码

### 13. Spring Framework
- 13.1 Spring IoC 容器与 Bean 生命周期
- 13.2 依赖注入（DI）的实现原理
- 13.3 AOP 与动态代理（JDK Proxy / CGLIB）
- 13.4 事件机制（ApplicationEvent）
- 13.5 国际化与资源管理
- 13.6 Spring 循环依赖与三级缓存
- 13.7 FactoryBean 与 BeanFactory 区别
- 13.8 Spring 扩展点（BeanPostProcessor、BeanFactoryPostProcessor 等）

### 14. Spring Boot
- 14.1 自动装配原理（@EnableAutoConfiguration 源码）
- 14.2 Starter 机制与自定义 Starter
- 14.3 Spring Boot 启动流程详解
- 14.4 条件注解（@Conditional 系列）
- 14.5 配置加载机制与多环境管理
- 14.6 Actuator 监控与健康检查

### 15. Spring MVC
- 15.1 请求处理全流程（DispatcherServlet 源码）
- 15.2 HandlerMapping 与 HandlerAdapter
- 15.3 参数解析器与返回值处理器
- 15.4 拦截器 vs 过滤器
- 15.5 全局异常处理（@ControllerAdvice）
- 15.6 文件上传与下载

### 16. MyBatis
- 16.1 ORM 思想与 MyBatis 架构
- 16.2 核心配置与映射文件
- 16.3 动态 SQL 与常用标签
- 16.4 缓存机制（一级缓存、二级缓存）
- 16.5 插件机制（Interceptor）与 PageHelper
- 16.6 MyBatis-Plus 快速开发

---

## 第五阶段：微服务架构

### 17. 微服务理论
- 17.1 单体 → SOA → 微服务演进
- 17.2 微服务设计原则与拆分策略
- 17.3 服务治理全景图
- 17.4 CAP 理论与 BASE 理论
- 17.5 DDD 与微服务结合实践

### 18. Spring Cloud 生态
- 18.1 服务注册与发现（Nacos / Eureka / Consul）
- 18.2 服务调用（OpenFeign / RestTemplate）
- 18.3 负载均衡（Spring Cloud LoadBalancer / Ribbon）
- 18.4 服务熔断与降级（Sentinel / Resilience4j / Hystrix）
- 18.5 网关（Spring Cloud Gateway / Zuul）
- 18.6 配置中心（Nacos Config / Apollo）
- 18.7 链路追踪（SkyWalking / Zipkin / Sleuth）
- 18.8 分布式事务（Seata AT/TCC/Saga 模式）

### 19. 消息中间件
- 19.1 消息队列核心概念（生产者、消费者、Topic、Queue）
- 19.2 RocketMQ（架构、顺序消息、延迟消息、事务消息）
- 19.3 Kafka（分区、副本、消费者组、Exactly-Once）
- 19.4 RabbitMQ（交换机类型、死信队列、延迟队列）
- 19.5 消息可靠性保证（生产端、Broker、消费端）
- 19.6 消息积压处理方案

### 20. RPC 与服务通信
- 20.1 RPC 原理与手写简易 RPC 框架
- 20.2 Dubbo 核心架构与使用
- 20.3 gRPC 与 Protobuf
- 20.4 序列化对比（JSON、Hessian、Protobuf、Kryo）

---

## 第六阶段：分布式与高可用

### 21. 分布式系统核心
- 21.1 分布式 ID 生成（雪花算法、Leaf、UUID）
- 21.2 分布式锁（Redis、ZooKeeper、数据库实现对比）
- 21.3 分布式会话（Session 共享、JWT、SSO）
- 21.4 分布式缓存设计（多级缓存、缓存一致性）
- 21.5 分布式定时任务（XXL-JOB、Elastic-Job）

### 22. ZooKeeper
- 22.1 ZAB 协议与选举机制
- 22.2 数据模型与 Watcher 机制
- 22.3 ZooKeeper 典型应用场景
- 22.4 ZooKeeper 与分布式协调

### 23. 高可用架构
- 23.1 高可用设计原则（冗余、限流、熔断、降级）
- 23.2 限流算法（令牌桶、漏桶、滑动窗口）
- 23.3 服务容错与重试策略
- 23.4 灰度发布与蓝绿部署
- 23.5 混沌工程与故障演练

---

## 第七阶段：性能优化

### 24. 应用性能优化
- 24.1 代码层面优化（减少对象创建、避免 N+1 查询、懒加载）
- 24.2 数据库性能调优（SQL 优化、连接池配置、批量操作）
- 24.3 缓存优化策略与热点 Key 处理
- 24.4 JVM 调优实战（GC 日志分析、堆内存调优）
- 24.5 线程池调优与监控

### 25. 高并发系统设计
- 25.1 秒杀系统设计
- 25.2 短链系统设计
- 25.3 Feed 流系统设计
- 25.4 评论系统设计
- 25.5 排行榜系统设计
- 25.6 大文件上传与分片下载

---

## 第八阶段：DevOps 与工程化

### 26. 构建与版本管理
- 26.1 Maven 核心（依赖管理、生命周期、插件开发）
- 26.2 Gradle 入门与实战
- 26.3 Git 工作流（Git Flow、Trunk-Based）
- 26.4 代码规范与 Checkstyle / SpotBugs

### 27. 容器化与云原生
- 27.1 Docker 基础（镜像、容器、Dockerfile、Compose）
- 27.2 Kubernetes 核心概念（Pod、Service、Deployment、Ingress）
- 27.3 Helm Chart 与应用部署
- 27.4 服务网格（Istio / Linkerd）入门
- 27.5 云原生 12 要素与应用设计

### 28. CI/CD 与监控
- 28.1 Jenkins / GitHub Actions 流水线
- 28.2 日志体系（ELK / EFK Stack）
- 28.3 监控告警（Prometheus + Grafana）
- 28.4 APM 与分布式链路追踪（SkyWalking / Pinpoint）

---

## 第九阶段：架构师视野

### 29. 软件架构设计
- 29.1 架构风格（分层、微内核、事件驱动、Serverless）
- 29.2 架构设计方法论（4+1 视图、C4 模型）
- 29.3 技术选型与权衡决策
- 29.4 架构评审与治理
- 29.5 技术债务管理

### 30. 中间件设计
- 30.1 手写简易版 Spring IoC 容器
- 30.2 手写简易版 RPC 框架
- 30.3 手写简易版消息队列
- 30.4 手写简易版连接池
- 30.5 手写简易版网关

### 31. 大数据与 AI 协作
- 31.1 Java 与大数据生态（Hadoop、Spark、Flink 入门）
- 31.2 Java 接入 AI 能力（LangChain4j / Spring AI）
- 31.3 向量数据库与 RAG 检索增强生成

---

## 附录

### A. 开发工具链
- A.1 IntelliJ IDEA 高效使用技巧
- A.2 终端工具与命令行效率
- A.3 抓包工具（Wireshark、Charles）
- A.4 接口调试（Postman、cURL）

### B. 面试与成长
- B.1 简历优化与技术亮点提炼
- B.2 技术面试高频考点梳理
- B.3 系统设计面试方法论
- B.4 技术影响力打造（开源、博客、分享）

### C. 推荐书单与资源
- C.1 Java 基础进阶书籍
- C.2 JVM 与并发经典书籍
- C.3 架构设计经典书籍
- C.4 优质技术社区与博客
