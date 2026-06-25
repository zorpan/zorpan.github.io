---
title: Spring Framework 深度解析：IoC、AOP 与 Bean 生命周期
date: 2026-06-17 10:00:00
tags:
  - Spring
  - IoC
  - AOP
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 13.1 Spring IoC 容器与 Bean 生命周期

### 13.1.1 IoC 核心思想

IoC（控制反转）：将对象的创建和依赖管理交给 Spring 容器，而不是由程序员手动 new 对象。

```
传统方式：程序员手动创建和组装对象
UserDao dao = new UserDao();
UserService service = new UserService(dao);  // 手动注入

IoC 方式：Spring 容器自动创建和注入
@Autowired
private UserService service;  // 容器自动注入
```

### 13.1.2 Bean 的作用域

| 作用域 | 说明 | 场景 |
|--------|------|------|
| `singleton` | 默认，整个容器中只有一个实例 | 无状态 Bean |
| `prototype` | 每次获取都创建新实例 | 有状态 Bean |
| `request` | 每个 HTTP 请求一个实例 | Web 应用 |
| `session` | 每个 HTTP Session 一个实例 | Web 应用 |

### 13.1.3 Bean 的完整生命周期

```
1. 实例化（Instantiation）
   └── 通过反射调用构造方法创建 Bean 实例

2. 属性填充（Populate Properties）
   └── @Autowired、@Value、@Resource 注入依赖

3. Aware 回调
   ├── BeanNameAware → setBeanName()
   ├── BeanFactoryAware → setBeanFactory()
   └── ApplicationContextAware → setApplicationContext()

4. BeanPostProcessor 前置处理
   └── postProcessBeforeInitialization()

5. 初始化（Initialization）
   ├── @PostConstruct 注解方法
   ├── InitializingBean → afterPropertiesSet()
   └── 自定义 init-method

6. BeanPostProcessor 后置处理
   └── postProcessAfterInitialization()
   └── AOP 代理在此处创建

7. Bean 就绪，可以使用

8. 销毁（Destroy）
   ├── @PreDestroy 注解方法
   ├── DisposableBean → destroy()
   └── 自定义 destroy-method
```

> **面试考点：BeanPostProcessor 在哪里用到？**
> AOP 代理创建（AbstractAutoProxyCreator）、@Autowired 注入（AutowiredAnnotationBeanPostProcessor）、@Async 异步处理（AsyncAnnotationBeanPostProcessor）等都在 BeanPostProcessor 中实现。它是 Spring 最重要的扩展机制之一。

---

## 13.2 依赖注入（DI）的实现原理

### 13.2.1 注入方式

```java
// 1. 构造器注入（推荐）
@Service
public class UserService {
    private final UserDao userDao;

    @Autowired  // Spring 4.3+ 单构造器可省略
    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}

// 2. 字段注入（不推荐，但最常用）
@Service
public class UserService {
    @Autowired
    private UserDao userDao;
}

// 3. Setter 注入
@Service
public class UserService {
    private UserDao userDao;

    @Autowired
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

### 13.2.2 @Autowired 按类型注入流程

```
1. 按类型查找 → 找到 1 个 → 直接注入
2. 找到多个 → 按 @Qualifier 或变量名匹配
3. 匹配失败 → 抛 NoUniqueBeanDefinitionException
4. 找到 0 个 → required=true 抛 NoSuchBeanDefinitionException
```

### 13.2.3 @Autowired vs @Resource

| 特性 | @Autowired | @Resource |
|------|-----------|----------|
| 来源 | Spring | JSR-250（JDK） |
| 注入方式 | 按类型 | 先按名称，找不到再按类型 |
| 指定名称 | @Qualifier | name 属性 |
| 推荐 | Spring 生态内 | 需要按名称注入时 |

---

## 13.3 AOP 与动态代理

### 13.3.1 AOP 核心概念

| 概念 | 说明 |
|------|------|
| 切面（Aspect） | 横切关注点的模块化（如日志、事务） |
| 连接点（Join Point） | 程序执行的某个点（Spring AOP 只支持方法级） |
| 切入点（Pointcut） | 匹配连接点的表达式 |
| 通知（Advice） | 在切入点执行的动作 |
| 织入（Weaving） | 将切面应用到目标对象的过程 |

### 13.3.2 五种通知类型

```java
@Aspect
@Component
public class LogAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void before(JoinPoint jp) {
        // 前置通知：方法执行前
    }

    @After("execution(* com.example.service.*.*(..))")
    public void after(JoinPoint jp) {
        // 后置通知：方法执行后（无论是否异常）
    }

    @AfterReturning(pointcut = "execution(...)", returning = "result")
    public void afterReturning(Object result) {
        // 返回通知：方法正常返回后
    }

    @AfterThrowing(pointcut = "execution(...)", throwing = "ex")
    public void afterThrowing(Exception ex) {
        // 异常通知：方法抛异常后
    }

    @Around("execution(* com.example.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        // 环绕通知：最强大，可以控制是否执行目标方法
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();  // 执行目标方法
        log.info("cost: {}ms", System.currentTimeMillis() - start);
        return result;
    }
}
```

**通知执行顺序：**
- 正常执行：@Around 前半段 → @Before → 目标方法 → @AfterReturning → @After → @Around 后半段
- 异常执行：@Around 前半段 → @Before → 目标方法(异常) → @AfterThrowing → @After → @Around 后半段

### 13.3.3 AOP 代理创建时机

```
BeanPostProcessor.postProcessAfterInitialization()
    └── AbstractAutoProxyCreator.wrapIfNecessary()
        ├── 判断是否需要代理（匹配切点）
        ├── 默认使用 CGLIB 代理（Spring Boot 2.x+）
        └── 仅在 proxyTargetClass=false 时使用 JDK 动态代理（需有接口）
```

---

## 13.4 Spring 循环依赖与三级缓存

### 13.4.1 什么是循环依赖

```java
@Service
public class A {
    @Autowired
    private B b;
}

@Service
public class B {
    @Autowired
    private A a;
}
```

### 13.4.2 三级缓存

```java
// DefaultSingletonBeanRegistry 中的三级缓存

// 一级缓存：完整的 Bean 实例（已初始化）
Map<String, Object> singletonObjects = new ConcurrentHashMap<>();

// 二级缓存：早期暴露的 Bean（已实例化，未初始化），用于解决循环依赖
Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>();

// 三级缓存：Bean 的工厂（ObjectFactory），用于延迟创建代理对象
Map<String, ObjectFactory<?>> singletonFactories = new ConcurrentHashMap<>();
```

### 13.4.3 解决流程（以 A → B → A 为例）

```
1. 创建 A：实例化 A → 将 A 的工厂放入三级缓存 → 填充属性发现依赖 B

2. 创建 B：实例化 B → 将 B 的工厂放入三级缓存 → 填充属性发现依赖 A

3. 获取 A：从三级缓存获取 A 的工厂 → 调用 getObject() 获取 A 的早期引用
   → 将 A 的早期引用放入二级缓存，移除三级缓存

4. B 完成属性填充和初始化 → 放入一级缓存

5. A 继续填充属性（此时 B 已就绪）→ 初始化完成 → 放入一级缓存
```

**为什么需要三级缓存而不是两级？**
第三级缓存的 `ObjectFactory` 可以延迟创建代理对象。如果 A 需要被 AOP 代理，只有在真正被依赖时才创建代理，而不是在实例化时就创建。这保证了代理对象的正确性。

> **面试考点：什么情况下循环依赖无法解决？**
> 1. 构造器注入的循环依赖无法解决（实例化时就需要依赖）
> 2. Prototype 作用域的循环依赖无法解决（不走缓存）
> 3. @Async 导致的循环依赖可能有问题（因为 @Async 创建了新的代理）

---

## 13.5 FactoryBean 与 BeanFactory 区别

```java
// BeanFactory：Spring IoC 容器的根接口，负责创建和管理 Bean
BeanFactory factory = new ClassPathXmlApplicationContext("beans.xml");
User user = factory.getBean(User.class);

// FactoryBean：特殊的 Bean，可以自定义 Bean 的创建逻辑
public class MyFactoryBean implements FactoryBean<ComplexObject> {
    @Override
    public ComplexObject getObject() throws Exception {
        return new ComplexObject();  // 自定义创建逻辑
    }

    @Override
    public Class<?> getObjectType() {
        return ComplexObject.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
// getBean("myFactoryBean") → 返回 ComplexObject（不是 FactoryBean 本身）
// getBean("&myFactoryBean") → 返回 MyFactoryBean 实例
```

---

## 13.6 Spring 扩展点

### 13.6.1 BeanFactory 级别（容器级）

```java
// BeanDefinitionRegistryPostProcessor
// 在 Bean 定义注册后、实例化前执行，可以修改 Bean 定义
@Component
public class MyBeanDefinitionRegistryPostProcessor
    implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        // 动态注册 BeanDefinition
    }
}

// BeanFactoryPostProcessor
// 在 Bean 实例化前，可以修改 Bean 属性值
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        // 修改 BeanDefinition 的属性值
    }
}
```

### 13.6.2 Bean 级别

```java
// BeanPostProcessor（最常用）
// 在 Bean 初始化前后执行
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;  // AOP 代理在此创建
    }
}

// InstantiationAwareBeanPostProcessor
// 在 Bean 实例化前后执行（比 BeanPostProcessor 更早）

// DestructionAwareBeanPostProcessor
// 在 Bean 销毁前执行
```

---

## 13.7 事件机制（ApplicationEvent）

```java
// 定义事件
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;
    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
}

// 发布事件
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void createOrder(Order order) {
        // 创建订单...
        publisher.publishEvent(new OrderCreatedEvent(this, order));
    }
}

// 监听事件
@Component
public class OrderEventListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // 发送通知、更新库存等
    }

    @EventListener
    @Async  // 异步处理
    public void onOrderCreatedAsync(OrderCreatedEvent event) {
        // 异步处理
    }
}
```

---

## 面试精选

### 1. Spring IoC 的原理？

IoC（控制反转）将对象的创建和依赖管理交给 Spring 容器。核心是 BeanFactory/ApplicationContext，通过读取配置（XML/注解/Java Config）解析为 BeanDefinition，然后通过反射创建 Bean 实例，自动注入依赖。依赖注入方式有构造器注入、字段注入、Setter 注入。

### 2. Bean 的生命周期？

实例化（反射创建）→ 属性填充（注入依赖）→ Aware 回调 → BeanPostProcessor 前置 → 初始化（@PostConstruct → afterPropertiesSet → init-method）→ BeanPostProcessor 后置（AOP 代理创建）→ 使用 → 销毁（@PreDestroy → destroy → destroy-method）。

### 3. AOP 的实现原理？JDK 代理和 CGLIB 的区别？

Spring AOP 通过动态代理实现，在 BeanPostProcessor 后置处理阶段创建代理对象。有接口默认用 JDK 动态代理（基于接口），无接口用 CGLIB（基于继承）。Spring Boot 2.x+ 默认强制用 CGLIB。AOP 用于日志、事务、权限等横切关注点。

### 4. Spring 怎么解决循环依赖？

通过三级缓存。一级缓存存完整 Bean，二级缓存存早期引用（已实例化未初始化），三级缓存存 Bean 工厂（延迟创建代理）。创建 A 时发现依赖 B，先将 A 的工厂放入三级缓存，然后创建 B，B 从三级缓存获取 A 的早期引用完成注入，最后 A 继续初始化。只支持单例 Setter 注入的循环依赖。

### 5. 为什么需要三级缓存而不是两级？

第三级缓存的 ObjectFactory 可以延迟创建代理对象。如果 A 需要 AOP 代理，只有在被依赖时才通过 ObjectFactory 创建代理，而不是实例化时就创建。如果只有两级缓存，要么所有 Bean 都提前创建代理（不正确），要么无法处理需要代理的循环依赖。

### 6. @Autowired 和 @Resource 的区别？

@Autowired 是 Spring 注解，按类型注入，找到多个再按名称匹配，可配合 @Qualifier 指定。@Resource 是 JSR-250 注解，先按名称注入，名称找不到再按类型。推荐构造器注入，避免字段注入。

### 7. Spring 有哪些扩展点？

容器级：BeanDefinitionRegistryPostProcessor（动态注册 Bean）、BeanFactoryPostProcessor（修改 Bean 定义）。Bean 级：BeanPostProcessor（初始化前后处理，AOP 在此创建）、InstantiationAwareBeanPostProcessor（实例化前后）、@PostConstruct/@PreDestroy、InitializingBean/DisposableBean、Aware 接口。

### 8. FactoryBean 和 BeanFactory 的区别？

BeanFactory 是 IoC 容器的根接口，负责创建和管理所有 Bean。FactoryBean 是一个特殊的 Bean，可以自定义 Bean 的创建逻辑（getObject()）。getBean("name") 返回 FactoryBean 创建的对象，getBean("&name") 返回 FactoryBean 本身。MyBatis 的 Mapper 就是通过 FactoryBean 实现的。
