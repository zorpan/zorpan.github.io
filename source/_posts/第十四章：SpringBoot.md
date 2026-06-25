---
title: Spring Boot 深度解析：自动装配原理与启动流程
date: 2026-06-17 12:00:00
tags:
  - Spring Boot
  - 自动装配
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

# 第十四章：Spring Boot

> Spring Boot 通过自动装配和约定优于配置的理念，大幅简化了 Spring 应用的开发。理解自动装配原理和启动流程是面试高频考点。

---

## 14.1 自动装配原理

### 14.1.1 @SpringBootApplication

```java
@SpringBootApplication
// 等价于
@SpringBootConfiguration    // 标记为配置类
@EnableAutoConfiguration    // 开启自动装配
@ComponentScan              // 包扫描
public class Application { }
```

### 14.1.2 @EnableAutoConfiguration 流程

```
@EnableAutoConfiguration
    └── @Import(AutoConfigurationImportSelector.class)
        └── selectImports()
            └── getAutoConfigurationEntry()
                └── getCandidateConfigurations()
                    └── SpringFactoriesLoader.loadFactoryNames()
                        └── 读取 META-INF/spring.factories
```

**核心流程：**
1. `@EnableAutoConfiguration` 通过 `@Import` 导入 `AutoConfigurationImportSelector`
2. `AutoConfigurationImportSelector` 读取 `spring.factories` 中配置的自动配置类
3. 通过 `@Conditional` 系列注解过滤，只加载满足条件的配置类
4. 配置类通过 `@Bean` 注册 Bean 到容器

### 14.1.3 自定义 Starter

```java
// 自动配置类
@AutoConfiguration
@ConditionalOnClass(MyService.class)
@ConditionalOnProperty(prefix = "my", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties.getName());
    }
}

// 配置属性类
@ConfigurationProperties(prefix = "my")
public class MyProperties {
    private String name = "default";
    // getter/setter
}
```

---

## 14.2 条件注解（@Conditional 系列）

| 条件注解 | 说明 |
|---------|------|
| `@ConditionalOnClass` | 类路径中存在指定类时生效 |
| `@ConditionalOnMissingClass` | 类路径中不存在指定类时生效 |
| `@ConditionalOnBean` | 容器中存在指定 Bean 时生效 |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean 时生效 |
| `@ConditionalOnProperty` | 配置属性满足条件时生效 |
| `@ConditionalOnWebApplication` | Web 应用时生效 |

---

## 14.3 Spring Boot 启动流程详解

```
SpringApplication.run() 启动流程：

1. 创建 SpringApplication 实例
   ├── 推断应用类型（SERVLET / REACTIVE / NONE）
   ├── 加载 SpringApplicationRunListener
   └── 加载 ApplicationContextInitializer

2. 执行 run() 方法
   ├── 准备环境变量（Environment）
   │   └── 加载 application.properties / application.yml
   ├── 创建 ApplicationContext（容器）
   ├── 准备容器（prepareContext）
   │   └── 加载主配置类
   ├── 刷新容器（refreshContext）← 核心步骤
   │   └── 创建嵌入式 Web 服务器（Tomcat/Jetty/Undertow）
   │   └── 实例化所有非懒加载的单例 Bean
   ├── 调用 Runner（CommandLineRunner / ApplicationRunner）
   └── 返回 ApplicationContext
```

---

## 14.4 配置加载机制与多环境管理

### 配置加载优先级（从高到低）

```
1. 命令行参数 --server.port=8080
2. Java 系统属性 -Dserver.port=8080
3. 操作系统环境变量 SERVER_PORT=8080
4. application-{profile}.yml（激活的 profile）
5. application.yml
6. @Configuration 类上的 @PropertySource
7. 默认属性
```

---

## 14.5 Actuator 监控与健康检查

常用端点：
- `/actuator/health`：健康检查
- `/actuator/metrics`：指标数据
- `/actuator/beans`：所有 Bean
- `/actuator/mappings`：所有请求映射

---

## 面试精选

### 1. Spring Boot 自动装配的原理？

@EnableAutoConfiguration 通过 @Import 导入 AutoConfigurationImportSelector，读取 META-INF/spring.factories 中的自动配置类列表，然后通过 @Conditional 系列注解过滤，只加载满足条件的配置类，配置类通过 @Bean 注册 Bean 到容器。

### 2. @SpringBootApplication 包含哪些注解？

三个核心注解：@SpringBootConfiguration（标记配置类）、@EnableAutoConfiguration（开启自动装配）、@ComponentScan（包扫描）。

### 3. Spring Boot 的启动流程？

创建 SpringApplication → 推断应用类型 → 加载 Listener 和 Initializer → 准备 Environment → 创建 ApplicationContext → 刷新容器 → 创建嵌入式 Web 服务器 → 实例化所有 Bean → 调用 Runner → 启动完成。

### 4. 怎么自定义一个 Starter？

1）创建 @ConfigurationProperties 配置属性类；2）创建 @AutoConfiguration + @Conditional 自动配置类；3）在 META-INF/spring.factories 中注册；4）打成 jar 引入依赖即可。

### 5. @Conditional 注解的作用？

条件装配核心，满足条件时配置类/Bean 才生效。常用：@ConditionalOnClass、@ConditionalOnMissingBean、@ConditionalOnProperty。Spring Boot 自动装配大量使用实现按需加载。
