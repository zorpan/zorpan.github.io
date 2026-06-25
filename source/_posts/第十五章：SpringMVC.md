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
    │          遍历所有 @RequestMapping 注解，匹配 URL 和 HTTP 方法
    │
    ├── 2. getHandlerAdapter() → 找到能执行该 Handler 的适配器
    │      └── HandlerAdapter（RequestMappingHandlerAdapter）
    │
    ├── 3. 执行拦截器 preHandle()
    │
    ├── 4. ha.handle() → 执行 Handler（Controller 方法）
    │      ├── 参数解析（HandlerMethodArgumentResolver）
    │      │   ├── @RequestParam → RequestParamMethodArgumentResolver
    │      │   ├── @RequestBody → RequestResponseBodyMethodProcessor
    │      │   ├── @PathVariable → PathVariableMethodArgumentResolver
    │      │   └── 自定义参数解析器
    │      └── 返回值处理（HandlerMethodReturnValueHandler）
    │          ├── @ResponseBody → RequestResponseBodyMethodProcessor → JSON 序列化
    │          └── 返回 ModelAndView → 视图解析
    │
    ├── 5. 执行拦截器 postHandle()（注：Handler 抛异常时不会执行，直接进入 afterCompletion）
    │
    └── 6. 执行拦截器 afterCompletion()（无论是否异常，所有 preHandle 返回 true 的拦截器都会执行）
```

---

## 15.2 HandlerMapping 与 HandlerAdapter

```java
// HandlerMapping：URL → Handler 的映射
// RequestMappingHandlerMapping：处理 @RequestMapping 注解

// HandlerAdapter：执行 Handler 的适配器
// RequestMappingHandlerAdapter：执行 @RequestMapping 标注的方法
// HttpRequestHandlerAdapter：执行 HttpRequestHandler
// SimpleControllerHandlerAdapter：执行 Controller 接口

// 设计模式：适配器模式
// DispatcherServlet 不直接调用各种 Handler，通过适配器统一接口
```

---

## 15.3 参数解析器

```java
@RestController
public class UserController {

    // @RequestParam：查询参数
    @GetMapping("/user")
    public User getUser(@RequestParam String name,
                        @RequestParam(defaultValue = "1") int page) { }

    // @PathVariable：路径参数
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) { }

    // @RequestBody：请求体（JSON → 对象）
    @PostMapping("/user")
    public User createUser(@RequestBody @Valid UserDTO user) { }

    // @RequestHeader：请求头
    @GetMapping("/info")
    public String info(@RequestHeader("Authorization") String token) { }

    // @ModelAttribute：表单参数（默认，不需要注解）
    @PostMapping("/form")
    public String form(UserDTO user) { }
}
```

---

## 15.4 拦截器 vs 过滤器

| 特性 | 过滤器（Filter） | 拦截器（Interceptor） |
|------|-----------------|---------------------|
| 规范 | Servlet 规范 | Spring MVC 框架 |
| 作用范围 | 所有请求（包括静态资源） | 只拦截 Controller 请求 |
| 执行时机 | DispatcherServlet 之前 | DispatcherServlet 之内 |
| 获取 Bean | 不能直接获取 | 可以注入 Spring Bean |
| 执行顺序 | Filter → Interceptor → Controller → Interceptor → Filter |

```java
// 拦截器实现
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null || !tokenService.validate(token)) {
            response.setStatus(401);
            return false;  // 返回 false 中断请求
        }
        return true;  // 返回 true 继续执行
    }

    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler,
                           ModelAndView modelAndView) { }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler,
                                Exception ex) { }
}

// 注册拦截器
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/login", "/api/register");
    }
}
```

---

## 15.5 全局异常处理（@ControllerAdvice）

```java
@RestControllerAdvice  // = @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    // 业务异常
    @ExceptionHandler(BizException.class)
    public Result<?> handleBizException(BizException e) {
        log.warn("业务异常: {}", e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }

    // 参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<?> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return Result.fail(400, message);
    }

    // 兜底异常
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

客户端请求 → Tomcat → DispatcherServlet.doDispatch() → HandlerMapping 找到 Controller 方法 → HandlerAdapter 执行方法（参数解析 + 业务逻辑 + 返回值处理）→ 视图渲染或 JSON 序列化 → 响应客户端。拦截器在 Handler 执行前后介入。

### 2. 拦截器和过滤器的区别？

过滤器是 Servlet 规范，在 DispatcherServlet 之前执行，作用于所有请求包括静态资源。拦截器是 Spring MVC 框架，在 DispatcherServlet 之内执行，只拦截 Controller 请求。拦截器可以注入 Spring Bean，过滤器不能。执行顺序：Filter → Interceptor → Controller → Interceptor → Filter。

### 3. @ControllerAdvice 的作用？

@ControllerAdvice 是 Spring MVC 的全局异常处理、数据绑定和数据预处理的增强注解。配合 @ExceptionHandler 处理全局异常，配合 @InitBinder 自定义参数绑定，配合 @ModelAttribute 全局数据预处理。@RestControllerAdvice = @ControllerAdvice + @ResponseBody。

### 4. @RequestBody 的原理？

@RequestBody 通过 RequestResponseBodyMethodProcessor 处理，使用 HttpMessageConverter（默认 Jackson 的 MappingJackson2HttpMessageConverter）将请求体的 JSON 反序列化为 Java 对象。配合 @Valid 注解可进行参数校验。

### 5. DispatcherServlet 的作用？

DispatcherServlet 是 Spring MVC 的前端控制器，所有请求的统一入口。核心方法 doDispatch() 负责：查找 Handler、获取 HandlerAdapter、执行拦截器、调用 Handler 处理请求、处理返回值。它本身是一个 Servlet，配置在 web.xml 或由 Spring Boot 自动配置。
