---
title: Java 8~21 新特性全解析
date: 2026-06-14 20:00:00
tags:
  - Java
  - Lambda
  - Stream
  - 新特性
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 6.1 Lambda 表达式与函数式接口

### 6.1.1 Lambda 语法

```java
// 完整形式
Comparator<String> comp = (String a, String b) -> { return a.length() - b.length(); };

// 省略参数类型（编译器推断）
Comparator<String> comp = (a, b) -> a.length() - b.length();

// 单参数省略括号
list.forEach(item -> System.out.println(item));

// 无参数
Runnable r = () -> System.out.println("hello");
```

### 6.1.2 函数式接口

**只有一个抽象方法的接口**（可以用 `@FunctionalInterface` 标注）。

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);  // 唯一的抽象方法
    // 可以有默认方法和静态方法
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        return v -> apply(before.apply(v));
    }
}
```

**JDK 内置的常用函数式接口：**

| 接口 | 方法 | 说明 | 示例 |
|------|------|------|------|
| `Function<T, R>` | `R apply(T t)` | 转换 | `s -> s.length()` |
| `Predicate<T>` | `boolean test(T t)` | 判断 | `s -> s.isEmpty()` |
| `Consumer<T>` | `void accept(T t)` | 消费 | `s -> System.out.println(s)` |
| `Supplier<T>` | `T get()` | 生产 | `() -> new ArrayList<>()` |
| `BiFunction<T, U, R>` | `R apply(T t, U u)` | 双参转换 | `(a, b) -> a + b` |
| `UnaryOperator<T>` | `T apply(T t)` | 一元操作 | `s -> s.toUpperCase()` |

### 6.1.3 方法引用

```java
// 1. 静态方法引用 ClassName::staticMethod
Function<String, Integer> toInt = Integer::parseInt;

// 2. 实例方法引用（对象已知） instance::method
String str = "hello";
Supplier<String> upper = str::toUpperCase;

// 3. 实例方法引用（对象未知，第一个参数作为调用者） ClassName::method
BiFunction<String, String, Boolean> equals = String::equals;

// 4. 构造器引用 ClassName::new
Supplier<List<String>> listFactory = ArrayList::new;
Function<Integer, List<String>> sizedList = ArrayList::new;
```

---

## 6.2 Stream API

### 6.2.1 Stream 的特点

- **不存储数据**：Stream 不是数据结构，是对数据源的视图
- **不修改源**：操作产生新的 Stream
- **延迟执行**：中间操作是惰性的，遇到终止操作才执行
- **一次性**：Stream 被消费后不能重复使用

### 6.2.2 创建 Stream

```java
// 从集合创建
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> stream = list.stream();
Stream<String> parallelStream = list.parallelStream();

// 从数组创建
Stream<int[]> arrStream = Arrays.stream(new int[]{1, 2, 3});

// 直接创建
Stream<String> s = Stream.of("a", "b", "c");

// 生成无限流
Stream<Integer> naturals = Stream.iterate(0, n -> n + 1);  // 0,1,2,3...
Stream<Double> randoms = Stream.generate(Math::random);     // 随机数

// 基本类型流（避免装箱开销）
IntStream intStream = IntStream.range(1, 10);       // 1~9
IntStream intStream2 = IntStream.rangeClosed(1, 10); // 1~10
```

### 6.2.3 中间操作（延迟执行）

```java
// filter：过滤
list.stream().filter(s -> s.length() > 3);

// map：映射转换
list.stream().map(String::toUpperCase);

// flatMap：扁平化映射（一对多）
List<List<Integer>> nested = Arrays.asList(
    Arrays.asList(1, 2), Arrays.asList(3, 4));
List<Integer> flat = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList());  // [1, 2, 3, 4]

// sorted：排序
list.stream().sorted(Comparator.comparing(String::length));

// distinct：去重
list.stream().distinct();

// limit / skip：截取
list.stream().limit(5);   // 取前 5 个
list.stream().skip(2);    // 跳过前 2 个

// peek：调试用，不修改元素
list.stream().peek(System.out::println).collect(Collectors.toList());
```

### 6.2.4 终止操作

```java
// 遍历
list.stream().forEach(System.out::println);

// 收集为集合
List<String> result = list.stream().collect(Collectors.toList());
Set<String> set = list.stream().collect(Collectors.toSet());
String joined = list.stream().collect(Collectors.joining(", "));

// 聚合操作
long count = list.stream().filter(s -> s.startsWith("a")).count();
int sum = IntStream.range(1, 101).sum();           // 5050
OptionalInt max = IntStream.range(1, 101).max();

// reduce：归约
int product = IntStream.rangeClosed(1, 5).reduce(1, (a, b) -> a * b);  // 120

// 分组
Map<Integer, List<String>> grouped = list.stream()
    .collect(Collectors.groupingBy(String::length));

// 分区
Map<Boolean, List<String>> partitioned = list.stream()
    .collect(Collectors.partitioningBy(s -> s.length() > 3));

// 查找
Optional<String> first = list.stream().filter(s -> s.startsWith("a")).findFirst();
boolean anyMatch = list.stream().anyMatch(s -> s.contains("x"));
boolean allMatch = list.stream().allMatch(s -> s.length() > 0);
```

> **面试考点：Stream 的中间操作和终止操作？**
> 中间操作（filter、map、sorted 等）返回新的 Stream，是惰性的，不会立即执行。终止操作（collect、forEach、reduce 等）触发实际计算，遍历数据源并产生结果。一个 Stream 只能被消费一次。

---

## 6.3 Optional 优雅处理空值

### 6.3.1 创建 Optional

```java
Optional<String> opt1 = Optional.of("hello");        // 不能为 null，否则 NPE
Optional<String> opt2 = Optional.ofNullable(value);   // 可以为 null
Optional<String> opt3 = Optional.empty();              // 空的 Optional
```

### 6.3.2 使用 Optional

```java
// 获取值
String value = opt1.get();              // 如果为空抛 NoSuchElementException
String value = opt1.orElse("default");  // 为空返回默认值
String value = opt1.orElseGet(() -> computeDefault());  // 为空时惰性计算默认值
String value = opt1.orElseThrow(() -> new RuntimeException("值为空"));  // 为空抛异常

// 判断
opt1.isPresent();   // 是否有值
opt1.isEmpty();     // 是否为空（JDK 11+）
opt1.ifPresent(v -> System.out.println(v));  // 有值时执行
opt1.ifPresentOrElse(
    v -> System.out.println(v),
    () -> System.out.println("empty")
);

// 链式转换
Optional<Integer> length = Optional.ofNullable(name)
    .filter(n -> n.length() > 0)
    .map(String::length)
    .map(len -> len * 2);

// flatMap：避免嵌套 Optional
Optional<String> result = getUser(id)
    .flatMap(User::getAddress)
    .map(Address::getCity);
```

**最佳实践：**
- 不要对 Optional 调用 `get()` 而不先 `isPresent()`
- Optional 用作方法返回值，不要用作类字段、方法参数或集合元素
- 用 `map`/`flatMap` 链式处理替代嵌套 if-null 判断

---

## 6.4 接口默认方法与静态方法

```java
public interface Vehicle {
    // 抽象方法
    void start();

    // 默认方法（JDK 8+）
    default void honk() {
        System.out.println("beep beep!");
    }

    // 静态方法（JDK 8+）
    static Vehicle create() {
        return () -> System.out.println("starting");
    }
}

// 解决菱形继承问题
interface A {
    default void hello() { System.out.println("A"); }
}

interface B {
    default void hello() { System.out.println("B"); }
}

class C implements A, B {
    @Override
    public void hello() {
        A.super.hello();  // 显式选择 A 的实现
    }
}
```

---

## 6.5 Java 9~21 重要新特性概览

### 6.5.1 Java 9

```java
// 模块系统（Jigsaw）
module com.example.myapp {
    requires java.sql;
    exports com.example.api;
}

// 集合工厂方法（不可变集合）
List<String> list = List.of("a", "b", "c");
Map<String, Integer> map = Map.of("a", 1, "b", 2);
Set<Integer> set = Set.of(1, 2, 3);

// 接口私有方法
public interface MyInterface {
    private void helper() { }  // 接口内的私有方法
}

// 改进的 try-with-resources
InputStream is = new FileInputStream("test.txt");
try (is) {  // 变量可以在 try 外部声明（JDK 9+）
    // ...
}
```

### 6.5.2 Java 10

```java
// 局部变量类型推断（var）
var list = new ArrayList<String>();  // 推断为 ArrayList<String>
var stream = list.stream();          // 推断为 Stream<String>
var map = Map.of("a", 1);           // 推断为 Map<String, Integer>
// var 只能用于局部变量，不能用于方法参数、返回值、类字段
```

### 6.5.3 Java 11（LTS）

```java
// String 新增方法
" ".isBlank();                // true（只有空白字符）
" hello ".strip();            // "hello"（去除首尾空白，支持 Unicode）
"hello\nworld".lines();       // Stream<String>
"hello".repeat(3);            // "hellohellohello"

// Files 新增方法
String content = Files.readString(Path.of("test.txt"));
Files.writeString(Path.of("out.txt"), "content");

// var 可以用于 lambda 参数
list.sort((var a, var b) -> a.length() - b.length());
```

### 6.5.4 Java 14

```java
// switch 表达式（正式版）
String result = switch (day) {
    case MONDAY, FRIDAY -> "工作日";
    case SATURDAY, SUNDAY -> "周末";
    default -> "其他";
};

// Record（JDK 14 预览 JEP 359，JDK 16 正式 JEP 395）
public record Point(int x, int y) { }
// 自动生成：构造方法、getter（x()、y()）、equals、hashCode、toString
Point p = new Point(1, 2);
p.x();  // 1
```

### 6.5.5 Java 15

```java
// 文本块（正式版）
String json = """
        {
            "name": "Tom",
            "age": 25
        }
        """;

// Sealed Classes（密封类，预览）
public sealed class Shape permits Circle, Rectangle, Triangle { }
public final class Circle extends Shape { }
public non-sealed class Rectangle extends Shape { }
```

### 6.5.6 Java 17（LTS）

```java
// Sealed Classes 正式版（JEP 409）
public sealed class Shape permits Circle, Rectangle, Triangle { }

// Pattern Matching for instanceof 已在 Java 16 正式（JEP 394），此处不再赘述

// Context-Specific Deserialization Filters（JEP 415）
// Foreign Function & Memory API（第二次预览，JEP 412）
```

### 6.5.7 Java 21（LTS）

```java
// Virtual Threads（虚拟线程）
Thread.startVirtualThread(() -> {
    System.out.println("I'm a virtual thread!");
});

// try (var executor = Executors.newVirtualThreadPerTaskExecutor()) { }

// Pattern Matching for switch（正式版）
String result = switch (obj) {
    case Integer i -> "int: " + i;
    case String s  -> "str: " + s;
    case null      -> "null";
    default        -> "other";
};

// Record Patterns（记录模式）
record Point(int x, int y) {}

static void printSum(Object obj) {
    if (obj instanceof Point(int x, int y)) {
        System.out.println(x + y);
    }
}

// Sequenced Collections
SequencedCollection<String> list = new ArrayList<>();
list.addFirst("a");
list.addLast("b");
list.getFirst();  // "a"
list.getLast();   // "b"
list.reversed();  // 反转视图
```

---

## 面试精选

### 1. Lambda 表达式的本质是什么？

Lambda 的本质是函数式接口的实例（匿名内部类的简写）。编译器通过 invokedynamic 指令在运行时生成实现类，而不是在编译期生成匿名内部类。Lambda 要求目标类型必须是函数式接口（只有一个抽象方法）。

### 2. Stream 的中间操作和终止操作的区别？

中间操作（filter、map、sorted）返回新 Stream，延迟执行，支持链式调用。终止操作（collect、forEach、reduce）触发实际计算，遍历数据源产生结果。一个 Stream 只能被消费一次。多次终止操作会抛 IllegalStateException。

### 3. 什么是函数式接口？有哪些常见的？

只有一个抽象方法的接口（可以有多个默认方法和静态方法），用 @FunctionalInterface 注解标注。常见的有：Function（转换）、Predicate（判断）、Consumer（消费）、Supplier（生产）、BiFunction（双参转换）、UnaryOperator（一元操作）。

### 4. Optional 的正确用法？和 null 的区别？

Optional 是容器对象，用于优雅处理可能为 null 的值。正确用法：作为方法返回值（替代返回 null）、用 map/flatMap 链式处理、用 orElse/orElseThrow 处理空值。不要用 Optional 作为类字段、方法参数或集合元素。Optional 不是 null 的替代品，而是让"可能为空"这个语义更显式。

### 5. Java 8 的 Stream 和集合有什么区别？

集合是数据容器，存储数据，可以多次遍历；Stream 是对数据源的视图，不存储数据，只能消费一次。集合立即执行所有操作；Stream 支持延迟执行（惰性求值），中间操作不会立即计算。Stream 支持并行处理（parallelStream），底层使用 ForkJoinPool。

### 6. Record 是什么？有什么限制？

Record 是 Java 16 引入的特殊类，用于建模不可变数据。语法 `record Point(int x, int y) {}`，自动生成构造方法、getter（方法名与字段名相同）、equals、hashCode、toString。限制：隐式 final 不能被继承、不能声明实例字段、不能定义 compact 构造器以外的构造方法时给字段赋值。

### 7. Virtual Threads 是什么？和传统线程有什么区别？

虚拟线程（Java 21 正式）是 JVM 管理的轻量级线程，不绑定操作系统线程。创建成本极低（虚拟线程没有固定栈空间，初始状态极小 vs 平台线程约占 1MB 栈空间），可以创建数百万个。虚拟线程在阻塞操作时会自动释放底层平台线程，适合 IO 密集型场景。用 `Thread.startVirtualThread()` 或 `Executors.newVirtualThreadPerTaskExecutor()` 创建。
