---
title: Spring MVC 深度解析：请求处理流程与拦截器
date: 2026-06-17 14:00:00
tags:
  - Spring MVC
  - DispatcherServlet
  - 拦截器
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

# 第十五章：Spring MVC

> Spring MVC 是基于 Servlet 的 Web 框架，DispatcherServlet 是其核心。理解请求处理全流程和参数解析机制是 Web 开发面试的重点。

---

## 15.1 请求处理全流程

```
客户端请求
    │
    ▼
Tomcat 接收请求
    │
    ▼
DispatcherServlet.doDispatch()  ← 核心入口
    │
    ├── 1. getHandler() → 根据 URL 找到 Handler（Controller 方法）
    │      └── HandlerMapping（RequestMappingHandlerMapping）
    │
    ├── 2. getHandlerAdapter() → 找到能执行该 Handler 的适配器
    │      └── HandlerAdapter（RequestMappingHandlerAdapter）
    │
    ├── 3. 执行拦截器 preHandle()
    │
    ├── 4. ha.handle() → 执行 Handler（Controller 方法）
    │      ├── 参数解析（HandlerMethodArgumentResolver）
    │      └── 返回值处理（HandlerMethodReturnValueHandler）
    │
    ├── 5. 执行拦截器 postHandle()
    │
    └── 6. 执行拦截器 afterCompletion()
```

---

## 15.2 参数解析器

```java
@RestController
public class UserController {

    @GetMapping("/user")
    public User getUser(@RequestParam String name,
                        @RequestParam(defaultValue = "1") int page) { }

    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) { }

    @PostMapping("/user")
    public User createUser(@RequestBody @Valid UserDTO user) { }
}
```

---

## 15.3 拦截器 vs 过滤器

| 特性 | 过滤器（Filter） | 拦截器（Interceptor） |
|------|-----------------|---------------------|
| 规范 | Servlet 规范 | Spring MVC 框架 |
| 作用范围 | 所有请求（包括静态资源） | 只拦截 Controller 请求 |
| 执行时机 | DispatcherServlet 之前 | DispatcherServlet 之内 |
| 获取 Bean | 不能直接获取 | 可以注入 Spring Bean |

---

## 15.4 全局异常处理（@ControllerAdvice）

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public Result<?> handleBizException(BizException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<?> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return Result.fail(400, message);
    }

    @ExceptionHandler(Exception.class)
    public Result<?> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(500, "系统内部错误");
    }
}
```

---

## 面试精选

### 1. Spring MVC 的请求处理流程？

客户端请求 → Tomcat → DispatcherServlet.doDispatch() → HandlerMapping 找到 Controller 方法 → HandlerAdapter 执行方法 → 视图渲染或 JSON 序列化 → 响应客户端。拦截器在 Handler 执行前后介入。

### 2. 拦截器和过滤器的区别？

过滤器是 Servlet 规范，在 DispatcherServlet 之前执行，作用于所有请求。拦截器是 Spring MVC 框架，在 DispatcherServlet 之内执行，只拦截 Controller 请求。拦截器可以注入 Spring Bean。

### 3. @RequestBody 的原理？

通过 RequestResponseBodyMethodProcessor 处理，使用 HttpMessageConverter（Jackson）将 JSON 反序列化为 Java 对象。配合 @Valid 进行参数校验。

### 4. @ControllerAdvice 的作用？

全局异常处理（@ExceptionHandler）、数据绑定（@InitBinder）、数据预处理（@ModelAttribute）。@RestControllerAdvice = @ControllerAdvice + @ResponseBody。

### 5. DispatcherServlet 的作用？

Spring MVC 前端控制器，所有请求的统一入口。核心方法 doDispatch() 负责查找 Handler、获取适配器、执行拦截器、调用 Handler、处理返回值。
