---
title: MyBatis 深度解析：执行流程、动态 SQL 与插件机制
date: 2026-06-17 16:00:00
tags:
  - MyBatis
  - ORM
  - SQL
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

# 第十六章：MyBatis

> MyBatis 是最流行的半自动 ORM 框架，灵活的 SQL 编写和强大的插件机制使其在互联网公司广泛使用。理解其核心原理是后端开发的必备技能。

---

## 16.1 ORM 思想与 MyBatis 架构

### 16.1.1 三种 ORM 方式对比

| 框架 | 类型 | 特点 |
|------|------|------|
| JDBC | 手动 | 最底层，代码繁琐 |
| MyBatis | 半自动 | SQL 与代码分离，灵活控制 SQL |
| Hibernate/JPA | 全自动 | 面向对象操作，自动生成 SQL |

### 16.1.2 MyBatis 核心架构

```
MyBatis 执行流程：

1. 读取配置文件 → 解析为 Configuration 对象
2. 创建 SqlSessionFactory
3. 获取 SqlSession（一次会话，非线程安全）
4. 执行 SQL：
   ├── Mapper 代理（MapperProxy）
   ├── Executor（执行器）
   ├── StatementHandler：处理 JDBC Statement
   ├── ParameterHandler：设置参数
   └── ResultSetHandler：处理结果集
5. 返回结果
```

---

## 16.2 #{} 和 ${} 的区别

- `#{}`：预编译参数，使用 `?` 占位符，防 SQL 注入（推荐）
- `${}`：字符串拼接，直接替换，有 SQL 注入风险（只用于动态表名、列名）

---

## 16.3 动态 SQL

```xml
<!-- where：自动去除多余的 AND/OR -->
<select id="findUsers" resultType="User">
    SELECT * FROM user
    <where>
        <if test="name != null">AND user_name = #{name}</if>
        <if test="age != null">AND age = #{age}</if>
    </where>
</select>

<!-- foreach：遍历集合 -->
<select id="findByIds" resultType="User">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>

<!-- set：动态更新 -->
<update id="updateUser">
    UPDATE user
    <set>
        <if test="name != null">user_name = #{name},</if>
        <if test="age != null">age = #{age},</if>
    </set>
    WHERE id = #{id}
</update>
```

---

## 16.4 缓存机制

**一级缓存（SqlSession 级别）：** 默认开启，同一 SqlSession 中相同查询直接返回缓存。执行增删改或 clearCache() 会清空。

**二级缓存（namespace 级别）：** 需手动开启，跨 SqlSession 共享。分布式环境下容易脏数据，生产通常用 Redis 替代。

---

## 16.5 插件机制（Interceptor）

```java
@Intercepts({
    @Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class}
    )
})
public class SqlInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler handler = (StatementHandler) invocation.getTarget();
        String sql = handler.getBoundSql().getSql();
        log.info("执行 SQL: {}", sql);
        return invocation.proceed();
    }
}
```

**PageHelper 分页原理：** 拦截 Executor.query()，SQL 执行前拼接 LIMIT，执行后封装为 Page 对象。

---

## 16.6 MyBatis-Plus 快速开发

```java
// Mapper 继承 BaseMapper，自动拥有 CRUD
public interface UserMapper extends BaseMapper<User> { }

// Lambda 查询（推荐，避免字段名写错）
userMapper.selectList(new LambdaQueryWrapper<User>()
    .eq(User::getAge, 25)
    .like(User::getUserName, "张")
    .orderByDesc(User::getCreateTime));

// 分页
Page<User> page = new Page<>(1, 10);
userMapper.selectPage(page, queryWrapper);
```

---

## 面试精选

### 1. MyBatis 的执行流程？

读取配置 → SqlSessionFactory → SqlSession → Mapper 代理 → Executor → StatementHandler → ParameterHandler → 执行 SQL → ResultSetHandler → 返回结果。

### 2. #{} 和 ${} 的区别？

#{} 预编译参数，? 占位符，防 SQL 注入。${} 字符串拼接，有注入风险，只用于动态表名列名。

### 3. 一级缓存和二级缓存？

一级缓存 SqlSession 级别，默认开启，增删改清空。二级缓存 namespace 级别，需手动开启，跨 SqlSession 共享，分布式环境用 Redis 替代。

### 4. MyBatis 插件机制？

通过动态代理拦截四大对象的方法。@Intercepts 指定拦截目标，intercept() 实现拦截逻辑。PageHelper 分页插件拦截 Executor.query() 拼接 LIMIT。

### 5. MyBatis 和 JPA 的区别？

MyBatis 半自动，SQL 写在 XML/注解中，灵活可控，适合复杂查询。JPA 全自动，面向对象操作，自动生 SQL，适合简单 CRUD。互联网公司偏爱 MyBatis。

### 6. MyBatis-Plus 的增强？

BaseMapper 自动 CRUD、条件构造器、分页插件、代码生成器、逻辑删除、自动填充、乐观锁。只做增强不做改变，与 MyBatis 完全兼容。
