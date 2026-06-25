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

## 16.1 ORM 思想与 MyBatis 架构

### 16.1.1 三种 ORM 方式对比

| 框架 | 类型 | 特点 |
|------|------|------|
| JDBC | 手动 | 最底层，代码繁琐，需要手动处理结果集 |
| MyBatis | 半自动 | SQL 与代码分离，灵活控制 SQL |
| Hibernate/JPA | 全自动 | 面向对象操作，自动生成 SQL，灵活性差 |

### 16.1.2 MyBatis 核心架构

```
MyBatis 执行流程：

1. 读取配置文件（mybatis-config.xml / SqlSessionFactoryBean）
   └── 解析为 Configuration 对象

2. 创建 SqlSessionFactory
   └── DefaultSqlSessionFactory

3. 获取 SqlSession
   └── DefaultSqlSession（一次会话，非线程安全）

4. 执行 SQL
   ├── 通过 Mapper 接口代理（MapperProxy）
   │   └── MapperProxyFactory 创建代理对象
   ├── Executor（执行器）
   │   ├── SimpleExecutor：每次执行都创建 Statement
   │   ├── ReuseExecutor：复用 Statement
   │   ├── BatchExecutor：批量执行
   │   └── CachingExecutor：二级缓存装饰器，先查缓存再委托执行
   ├── StatementHandler：处理 JDBC Statement
   ├── ParameterHandler：设置参数
   └── ResultSetHandler：处理结果集

5. 返回结果
```

---

## 16.2 核心配置与映射文件

```xml
<!-- mybatis-config.xml -->
<configuration>
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>  <!-- 下划线转驼峰 -->
        <setting name="logImpl" value="SLF4J"/>                 <!-- 日志实现 -->
        <setting name="cacheEnabled" value="true"/>             <!-- 二级缓存 -->
    </settings>

    <typeAliases>
        <package name="com.example.entity"/>
    </typeAliases>

    <plugins>
        <plugin interceptor="com.github.pagehelper.PageInterceptor"/>
    </plugins>
</configuration>
```

```xml
<!-- Mapper XML -->
<mapper namespace="com.example.mapper.UserMapper">

    <resultMap id="userMap" type="User">
        <id property="id" column="id"/>
        <result property="userName" column="user_name"/>
        <result property="createTime" column="create_time"/>
    </resultMap>

    <select id="findById" resultMap="userMap">
        SELECT * FROM user WHERE id = #{id}
    </select>

    <select id="findByName" resultType="User">
        SELECT * FROM user WHERE user_name = #{name}
    </select>
</mapper>
```

**#{} 和 ${} 的区别：**
- `#{}`：预编译参数，使用 `?` 占位符，防 SQL 注入（推荐）
- `${}`：字符串拼接，直接替换，有 SQL 注入风险（只用于动态表名、列名）

---

## 16.3 动态 SQL 与常用标签

```xml
<!-- if：条件判断 -->
<select id="findUsers" resultType="User">
    SELECT * FROM user WHERE 1=1
    <if test="name != null and name != ''">
        AND user_name LIKE CONCAT('%', #{name}, '%')
    </if>
    <if test="age != null">
        AND age = #{age}
    </if>
</select>

<!-- where：自动去除多余的 AND/OR -->
<select id="findUsers" resultType="User">
    SELECT * FROM user
    <where>
        <if test="name != null">AND user_name = #{name}</if>
        <if test="age != null">AND age = #{age}</if>
    </where>
</select>

<!-- choose/when/otherwise：类似 switch -->
<select id="findUsers" resultType="User">
    SELECT * FROM user
    <where>
        <choose>
            <when test="name != null">AND user_name = #{name}</when>
            <when test="age != null">AND age = #{age}</when>
            <otherwise>AND status = 1</otherwise>
        </choose>
    </where>
</select>

<!-- foreach：遍历集合 -->
<select id="findByIds" resultType="User">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>

<!-- set：动态更新（自动去除多余逗号）-->
<update id="updateUser">
    UPDATE user
    <set>
        <if test="name != null">user_name = #{name},</if>
        <if test="age != null">age = #{age},</if>
    </set>
    WHERE id = #{id}
</update>

<!-- trim：自定义前缀后缀 -->
<trim prefix="SET" suffixOverrides=",">
    <if test="name != null">user_name = #{name},</if>
</trim>
```

---

## 16.4 缓存机制

### 16.4.1 一级缓存（SqlSession 级别）

```java
// 同一个 SqlSession 中，相同查询直接返回缓存
SqlSession session = sqlSessionFactory.openSession();
UserMapper mapper = session.getMapper(UserMapper.class);

User user1 = mapper.findById(1L);  // 查数据库
User user2 = mapper.findById(1L);  // 从一级缓存获取，不查数据库
// user1 == user2 → true

// 以下操作会清空一级缓存：
// 1. 执行 insert/update/delete
// 2. 调用 session.clearCache()
// 3. 不同的 SqlSession
```

### 16.4.2 二级缓存（Mapper/namespace 级别）

```xml
<!-- 开启二级缓存 -->
<cache
    eviction="LRU"           <!-- 淘汰策略 -->
    flushInterval="60000"    <!-- 刷新间隔（毫秒） -->
    size="1024"              <!-- 缓存对象数量 -->
    readOnly="true"/>        <!-- readOnly: true=返回同一引用(快但不安全); false=返回深拷贝(安全但有开销) -->
```

```java
// 二级缓存跨 SqlSession 共享（同一个 namespace）
SqlSession session1 = sqlSessionFactory.openSession();
UserMapper mapper1 = session1.getMapper(UserMapper.class);
mapper1.findById(1L);  // 查数据库，session1 关闭后写入二级缓存
session1.close();

SqlSession session2 = sqlSessionFactory.openSession();
UserMapper mapper2 = session2.getMapper(UserMapper.class);
mapper2.findById(1L);  // 从二级缓存获取
```

**注意：** 二级缓存在分布式环境下容易出现脏数据，生产环境通常关闭，使用 Redis 等外部缓存替代。

---

## 16.5 插件机制（Interceptor）

```java
// MyBatis 插件可以拦截四大对象的方法调用
// Executor、StatementHandler、ParameterHandler、ResultSetHandler

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
        BoundSql boundSql = handler.getBoundSql();
        String sql = boundSql.getSql();
        log.info("执行 SQL: {}", sql);

        // 修改 SQL
        // String newSql = sql + " LIMIT 1000";
        // 反射修改 BoundSql 中的 SQL
        // MetaObject metaObject = SystemMetaObject.forObject(boundSql);
        // metaObject.setValue("sql", newSql);

        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
}
```

**PageHelper 分页插件原理：**
拦截 Executor.query() 方法，改写 SQL 添加分页语法（MySQL 添加 LIMIT，Oracle 添加 ROWNUM 等），执行后将结果封装为 Page 对象。

```java
// 使用 PageHelper
PageHelper.startPage(1, 10);  // 设置分页参数（ThreadLocal）
List<User> users = mapper.findAll();  // SQL 自动追加 LIMIT
PageInfo<User> pageInfo = new PageInfo<>(users);
```

---

## 16.6 MyBatis-Plus 快速开发

```java
// 必须配置分页插件，否则分页不生效
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}

// 实体类
@Data
@TableName("user")
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String userName;
    private Integer age;
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
}

// Mapper 继承 BaseMapper，自动拥有 CRUD 方法
public interface UserMapper extends BaseMapper<User> { }

// 使用
userMapper.selectById(1L);
userMapper.selectList(new QueryWrapper<User>()
    .eq("age", 25)
    .like("user_name", "张")
    .orderByDesc("create_time"));
userMapper.insert(user);
userMapper.updateById(user);
userMapper.deleteById(1L);

// Lambda 查询（推荐，避免字段名写错）
userMapper.selectList(new LambdaQueryWrapper<User>()
    .eq(User::getAge, 25)
    .like(User::getUserName, "张"));

// 分页
Page<User> page = new Page<>(1, 10);
userMapper.selectPage(page, queryWrapper);
```

---

## 面试精选

### 1. MyBatis 的执行流程？

读取配置 → 创建 SqlSessionFactory → 获取 SqlSession → Mapper 代理（MapperProxy）→ Executor 执行器 → StatementHandler → ParameterHandler 设置参数 → 执行 SQL → ResultSetHandler 处理结果 → 返回。

### 2. #{} 和 ${} 的区别？

#{} 是预编译参数，使用 ? 占位符，防止 SQL 注入，MyBatis 会自动进行类型转换和转义。${} 是字符串直接拼接，有 SQL 注入风险，只用于动态表名、列名等不能用占位符的场景。

### 3. MyBatis 的一级缓存和二级缓存？

一级缓存是 SqlSession 级别，默认开启，同一 SqlSession 中相同查询直接返回缓存，执行增删改或 clearCache() 会清空。二级缓存是 namespace 级别，需要手动开启，跨 SqlSession 共享，但分布式环境下容易脏数据，生产通常用 Redis 替代。

### 4. MyBatis 的插件机制？怎么实现分页？

插件通过动态代理拦截四大对象（Executor、StatementHandler、ParameterHandler、ResultSetHandler）的方法调用。@Intercepts 指定拦截目标，intercept() 中实现拦截逻辑。PageHelper 分页插件拦截 Executor.query()，在 SQL 执行前拼接 LIMIT，执行后封装为 Page 对象。

### 5. MyBatis 和 JPA 的区别？

MyBatis 是半自动 ORM，SQL 写在 XML/注解中，灵活控制 SQL，适合复杂查询和性能优化。JPA（Hibernate）是全自动 ORM，面向对象操作，自动生成 SQL，适合简单 CRUD。互联网公司偏爱 MyBatis（SQL 可控），企业级应用偏爱 JPA（开发效率高）。

### 6. MyBatis-Plus 比 MyBatis 多了什么？

MyBatis-Plus 是 MyBatis 的增强工具，提供：BaseMapper 自动 CRUD 方法、条件构造器（QueryWrapper/LambdaQueryWrapper）、自动分页插件、代码生成器、逻辑删除、自动填充、乐观锁等。只做增强不做改变，与 MyBatis 完全兼容。
