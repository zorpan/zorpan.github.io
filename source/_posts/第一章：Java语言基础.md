---
title: Java 语言基础深入解析
date: 2026-06-14 10:00:00
tags:
  - Java
  - 基础
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 1.1 数据类型与变量

### 1.1.1 八种基本数据类型

| 类型 | 字节 | 范围 | 默认值 |
|------|------|------|--------|
| `byte` | 1 | -128 ~ 127 | 0 |
| `short` | 2 | -32768 ~ 32767 | 0 |
| `int` | 4 | -2^31 ~ 2^31-1 | 0 |
| `long` | 8 | -2^63 ~ 2^63-1 | 0L |
| `float` | 4 | IEEE 754 单精度 | 0.0f |
| `double` | 8 | IEEE 754 双精度 | 0.0d |
| `char` | 2 | 0 ~ 65535 (Unicode) | '' |
| `boolean` | 1/4 | true / false | false |

> **面试考点：boolean 占几个字节？**
> JVM 规范规定 boolean 数组中每个元素占 1 字节；对于非数组的单个 boolean，JVM 规范未做严格限定，但 HotSpot 实现中通常以 int 方式（4 字节）存储。实际大小取决于 JVM 实现。

### 1.1.2 包装类与自动装箱拆箱

```java
// 基本类型 → 包装类（自动装箱）
Integer a = 100;   // 编译器自动调用 Integer.valueOf(100)

// 包装类 → 基本类型（自动拆箱）
int b = a;         // 编译器自动调用 a.intValue()
```

**经典陷阱——Integer 缓存池：**

```java
Integer x = 127;
Integer y = 127;
System.out.println(x == y);  // true（缓存池范围 -128~127）

Integer m = 128;
Integer n = 128;
System.out.println(m == n);  // false（超出缓存池，创建了新对象）

System.out.println(m.equals(n)); // true（比较值）
```

`Integer.valueOf()` 源码中维护了一个缓存数组 `IntegerCache`，范围默认为 -128~127。这个范围可以通过 `-XX:AutoBoxCacheMax` JVM 参数调整。

### 1.1.3 基本类型转换

```java
// 隐式转换（小 → 大，自动）
int i = 100;
long l = i;     // int → long
double d = i;   // int → double

// 显式转换（大 → 小，可能丢失精度）
double pi = 3.14;
int n = (int) pi;  // 结果为 3，小数部分截断

long big = 300;
byte b = (byte) big;  // 结果为 44，只保留低 8 位
```

> **面试考点：`float f = 3.4;` 有什么问题？**
> 编译报错。`3.4` 默认是 double 类型，double 赋给 float 是大转小，需要强制转换：`float f = (float) 3.4;` 或 `float f = 3.4f;`

---

## 1.2 运算符与流程控制

### 1.2.1 易错运算符

```java
// 位运算 vs 逻辑运算
// & 和 | 可以用于 boolean，且不会短路
// && 和 || 会短路

// 自增陷阱
int i = 0;
i = i++;  // 结果 i = 0！
// 等价于：int temp = i; i = i + 1; i = temp;
// i++ 返回自增前的值，赋值运算将旧值写回 i

// 移位运算
// >>  算术右移（保留符号位）
// >>> 逻辑右移（高位补 0）
int a = -8;
System.out.println(a >> 1);   // -4
System.out.println(a >>> 1);  // 2147483644（很大的正数）
```

### 1.2.2 switch 语句

```java
// switch 支持的类型：byte, short, int, char, String（JDK 7+）, Enum
// 注意：case 后必须是常量表达式，不能是变量
switch (day) {
    case "MONDAY":
        System.out.println("周一");
        break;
    default:
        System.out.println("其他");
}
```

JDK 12 引入 switch 表达式预览（JEP 325），JDK 14 正式固化为标准特性（JEP 361）：

```java
String result = switch (day) {
    case "MONDAY", "FRIDAY" -> "工作日";
    case "SATURDAY", "SUNDAY" -> "周末";
    default -> {
        // 可以写多行代码
        yield "未知";
    }
};
```

---

## 1.3 数组与字符串

### 1.3.1 数组

```java
// 声明与初始化
int[] arr1 = new int[5];          // 默认值 0
int[] arr2 = {1, 2, 3, 4, 5};
int[] arr3 = new int[]{1, 2, 3};

// 二维数组（锯齿数组）
int[][] matrix = new int[3][];
matrix[0] = new int[1];
matrix[1] = new int[2];
matrix[2] = new int[3];
```

数组是对象，存储在堆中。`arr.length` 是数组的 final 字段，不是方法（注意与 String 的 `length()` 方法区分）。

### 1.3.2 String、StringBuilder、StringBuffer

**String 的不可变性：**

```java
public final class String {
    private final byte[] value;  // JDK 9+ 用 byte[] 代替 char[]
    private final byte coder;    // LATIN1(0) 或 UTF16(1)
}
```

- String 被 `final` 修饰，不可继承
- 内部 `byte[]` 数组被 `final` 修饰，引用不可变
- 不可变带来：线程安全、可缓存（字符串常量池）、hashCode 不变

**字符串常量池：**

```java
String s1 = "abc";           // 存入常量池
String s2 = "abc";           // 从常量池获取
String s3 = new String("abc"); // 堆上新建对象

System.out.println(s1 == s2);  // true（同一个常量池引用）
System.out.println(s1 == s3);  // false（不同对象）
System.out.println(s1.equals(s3)); // true（值相同）

// intern() 方法：将字符串放入常量池并返回池中引用
System.out.println(s3.intern() == s1); // true
```

**三者对比：**

| 特性 | String | StringBuilder | StringBuffer |
|------|--------|---------------|--------------|
| 可变性 | 不可变 | 可变 | 可变 |
| 线程安全 | 是（不可变） | 否 | 是（synchronized） |
| 性能 | 拼接慢 | 最快 | 较快 |
| 场景 | 少量操作 | 单线程拼接 | 多线程拼接 |

```java
// 字符串拼接的本质
String s = "a" + "b" + "c";
// 编译器优化为 String s = "abc";

String s = "";
for (int i = 0; i < 10000; i++) {
    s += i;  // 每次 += 都会创建新的 StringBuilder + toString()
}
// 应该改为：
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

> **面试考点：String 为什么设计成不可变？**
> 1. 安全性：作为参数传递时不会被修改（如网络连接、文件路径）
> 2. 线程安全：不可变对象天然线程安全
> 3. 常量池优化：相同字面量只存一份，节省内存
> 4. hashCode 缓存：只计算一次，适合作为 HashMap 的 key

---

## 1.4 方法定义与递归

### 1.4.1 参数传递机制

**Java 只有值传递，没有引用传递。**

```java
public static void main(String[] args) {
    int x = 10;
    changeValue(x);
    System.out.println(x);  // 仍然是 10

    StringBuilder sb = new StringBuilder("hello");
    changeRef(sb);
    System.out.println(sb);  // hello world

    changePoint(sb);
    System.out.println(sb);  // hello world（不是 hello）
}

static void changeValue(int a) {
    a = 20;  // 修改的是副本
}

static void changeRef(StringBuilder s) {
    s.append(" world");  // 修改的是引用指向的对象
}

static void changePoint(StringBuilder s) {
    s = new StringBuilder("hello");  // 修改的是引用本身，不影响原对象
}
```

- 基本类型：传递值的副本，修改不影响原值
- 引用类型：传递引用的副本（地址值的拷贝），可以通过副本修改对象内容，但不能改变原引用指向

### 1.4.2 方法重载（Overload）

```java
// 重载：同类中，方法名相同，参数列表不同（类型、个数、顺序）
// 与返回值无关！
public int add(int a, int b) { return a + b; }
public double add(double a, double b) { return a + b; }
public int add(int a, int b, int c) { return a + b + c; }
```

### 1.4.3 递归

```java
// 经典递归：斐波那契数列
public int fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
}

// 尾递归优化（Java 不支持，但概念重要）
// 尾递归：递归调用是函数的最后一个操作
public int fibTail(int n, int a, int b) {
    if (n == 0) return a;
    return fibTail(n - 1, b, a + b);  // 尾位置
}
```

> **面试考点：递归和迭代的区别？**
> 递归代码简洁但有调用栈开销，深度太大会 StackOverflowError；迭代性能更好但代码可能更复杂。能用迭代解决的问题优先用迭代，特别适合递归的场景（如树遍历）才用递归。

---

## 1.5 面向对象基础

### 1.5.1 封装、继承、多态

**封装：** 将数据和操作数据的方法绑定，隐藏实现细节。

```java
public class Account {
    private double balance;  // 私有字段

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException();
        balance += amount;
    }
}
```

**继承：** 子类继承父类的属性和方法（单继承）。

```java
public class Animal {
    public void eat() { System.out.println("eating"); }
}

public class Dog extends Animal {
    public void bark() { System.out.println("barking"); }
}
```

**多态：** 父类引用指向子类对象，运行时决定调用哪个方法。

```java
Animal animal = new Dog();  // 向上转型
animal.eat();   // 调用 Dog 的 eat（如果有重写）
animal.bark();  // 编译错误！编译看左边（Animal 类型没有 bark）

// 向下转型（需要先判断类型）
if (animal instanceof Dog) {
    Dog dog = (Dog) animal;  // 安全的向下转型
    dog.bark();
}
```

### 1.5.2 重写（Override）规则

```java
class Parent {
    protected Number doSomething(int x) { return x; }
}

class Child extends Parent {
    @Override
    // 方法签名必须相同
    // 返回值可以是父类返回值的子类（协变返回类型）
    // 访问权限不能更严格（可以更宽松）
    // 不能抛出更多/更大的受检异常
    public Integer doSomething(int x) { return x * 2; }
}
```

### 1.5.3 Object 类核心方法

```java
public class Object {
    // 1. equals()：对象相等性比较
    // 默认等同于 ==（比较引用地址），通常需要重写
    // 重写时必须同时重写 hashCode()

    // 2. hashCode()：返回对象的哈希值
    // equals 相等 → hashCode 必须相等
    // hashCode 相等 → equals 不一定相等

    // 3. toString()：返回对象的字符串表示
    // 默认：类名@hashCode十六进制

    // 4. clone()：对象克隆
    // 必须实现 Cloneable 接口，否则抛 CloneNotSupportedException

    // 5. finalize()：GC 前调用（已废弃，不推荐使用）

    // 6. getClass()：获取运行时类信息（反射）
}
```

**equals 和 hashCode 重写模板：**

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Person person = (Person) o;
    return age == person.age && Objects.equals(name, person.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```

> **面试考点：为什么重写 equals 必须重写 hashCode？**
> 因为 HashMap 等容器依赖 hashCode 定位桶。如果两个 equals 为 true 的对象 hashCode 不同，会被分配到不同的桶，导致 HashMap 中出现逻辑上重复的 key。规则：equals 相等 → hashCode 必须相等；hashCode 相等 → equals 不一定相等。

---

## 1.6 抽象类与接口

### 1.6.1 抽象类

```java
public abstract class Shape {
    protected String color;

    public Shape(String color) { this.color = color; }

    // 抽象方法：没有实现，子类必须实现
    public abstract double area();

    // 普通方法：可以有实现
    public String getColor() { return color; }
}
```

- 不能被实例化
- 可以有构造方法（供子类调用）
- 可以有普通方法和抽象方法
- 单继承

### 1.6.2 接口

```java
public interface Drawable {
    // 常量（默认 public static final）
    int MAX_SIZE = 100;

    // 抽象方法（默认 public abstract）
    void draw();

    // 默认方法（JDK 8+）
    default void print() {
        System.out.println("printing");
    }

    // 静态方法（JDK 8+）
    static Drawable create() {
        return () -> System.out.println("created");
    }

    // 私有方法（JDK 9+，用于默认方法之间的代码复用）
    private void helper() { }
}
```

### 1.6.3 抽象类 vs 接口

| 特性 | 抽象类 | 接口 |
|------|--------|------|
| 继承 | 单继承 | 多实现 |
| 构造方法 | 有 | 无 |
| 成员变量 | 可以有普通变量 | 只能有常量 |
| 方法 | 可以有普通方法 | 默认 public abstract |
| 设计目的 | 代码复用（is-a） | 行为契约（can-do） |
| 访问修饰符 | 任意 | 默认 public |

> **面试考点：什么时候用抽象类，什么时候用接口？**
> 当多个子类有共同的状态和行为时用抽象类（is-a 关系，如 Dog is Animal）；当需要定义一组行为规范、支持多继承时用接口（can-do 关系，如 Dog can Swim）。Java 8 以后接口能力增强，但本质区别不变：抽象类是模板，接口是契约。

---

## 1.7 内部类与匿名类

### 1.7.1 四种内部类

```java
public class Outer {
    private int x = 1;

    // 1. 成员内部类
    class MemberInner {
        void show() { System.out.println(x); }  // 可以访问外部类的私有成员
    }

    // 2. 静态内部类
    static class StaticInner {
        void show() { /* 不能访问 x，只能访问外部类的静态成员 */ }
    }

    // 3. 局部内部类（定义在方法中）
    void method() {
        class LocalInner {
            void show() { System.out.println(x); }
        }
        new LocalInner().show();
    }

    // 4. 匿名内部类
    Runnable r = new Runnable() {
        @Override
        public void run() { System.out.println("anonymous"); }
    };
}
```

**创建内部类实例：**

```java
Outer outer = new Outer();

// 成员内部类：需要外部类实例
Outer.MemberInner mi = outer.new MemberInner();

// 静态内部类：不需要外部类实例
Outer.StaticInner si = new Outer.StaticInner();
```

### 1.7.2 内部类与外部类的关系

- 成员内部类和局部内部类隐式持有外部类引用（可以导致内存泄漏）
- 静态内部类不持有外部类引用（推荐使用，避免内存泄漏）
- 匿名内部类不能定义构造方法

> **面试考点：内部类为什么能访问外部类的私有成员？**
> 编译器会生成一个合成方法（access method），通过这个方法间接访问私有成员。成员内部类编译后会生成 `Outer$Inner.class`，并持有一个 `this$0` 引用指向外部类实例。

---

## 1.8 枚举与注解

### 1.8.1 枚举

```java
public enum Season {
    SPRING("春天", 1),
    SUMMER("夏天", 2),
    AUTUMN("秋天", 3),
    WINTER("冬天", 4);

    private final String name;
    private final int order;

    Season(String name, int order) {
        this.name = name;
        this.order = order;
    }

    public String getName() { return name; }
}
```

枚举的本质是一个 `final class`，继承自 `java.lang.Enum`：
- 每个枚举值是一个 `public static final` 的实例
- 天然单例、线程安全
- 可以有构造方法、字段、方法
- 可以实现接口，但不能继承其他类

**枚举实现单例（推荐方式）：**

```java
public enum Singleton {
    INSTANCE;

    public void doSomething() { }
}
// 天然线程安全、防反射攻击、支持序列化
```

### 1.8.2 注解（Annotation）

```java
// 定义注解
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
    String value() default "";
    int priority() default 0;
}

// 使用注解
@MyAnnotation(value = "test", priority = 1)
public class MyClass { }

// 通过反射获取注解
MyAnnotation anno = MyClass.class.getAnnotation(MyAnnotation.class);
if (anno != null) {
    System.out.println(anno.value());    // "test"
    System.out.println(anno.priority()); // 1
}
```

**元注解说明：**

| 元注解 | 说明 |
|--------|------|
| `@Target` | 注解可以应用的位置（METHOD、TYPE、FIELD 等） |
| `@Retention` | 注解保留策略：SOURCE（编译丢弃）、CLASS（class文件保留）、RUNTIME（运行时可反射读取） |
| `@Documented` | 包含在 Javadoc 中 |
| `@Inherited` | 子类可以继承父类的注解 |
| `@Repeatable` | 允许在同一位置重复使用（JDK 8+） |

---

## 面试精选

### 1. Java 基本类型有哪些？String 是基本类型吗？

八种基本类型：byte、short、int、long、float、double、char、boolean。String 不是基本类型，是 `java.lang.String` 类，属于引用类型。String 被设计为不可变类，内部用 `final byte[]` 存储字符数据。

### 2. `==` 和 `equals()` 的区别？

`==` 比较的是引用地址（基本类型比较值）；`equals()` 默认等同于 `==`，但 String、Integer 等类重写了 `equals()` 方法，改为比较内容。重写 `equals()` 时必须同时重写 `hashCode()`，否则在 HashMap 等容器中会出现逻辑错误。

### 3. String、StringBuilder、StringBuffer 的区别？

String 不可变，每次拼接创建新对象；StringBuilder 可变，单线程下性能最好；StringBuffer 可变且线程安全（方法加 synchronized），多线程下使用。频繁拼接字符串应使用 StringBuilder，避免用 `+` 拼接。

### 4. 为什么重写 equals 必须重写 hashCode？

HashMap 先用 hashCode 定位桶，再用 equals 比较。如果两个 equals 为 true 的对象 hashCode 不同，会被放到不同桶中，导致 HashMap 中出现重复的逻辑等价 key。规则：equals 相等则 hashCode 必须相等。

### 5. Java 是值传递还是引用传递？

Java 只有值传递。基本类型传递值的副本，修改不影响原值；引用类型传递引用的副本（地址的拷贝），可以通过副本修改对象内容，但不能改变原引用的指向。

### 6. 抽象类和接口的区别？

抽象类单继承，可以有构造方法、普通方法和成员变量，适合代码复用（is-a）；接口多实现，只能有常量和抽象方法（JDK 8+ 支持默认方法），适合定义行为契约（can-do）。设计时优先考虑接口，需要共享状态或代码时用抽象类。

### 7. 什么是多态？底层怎么实现的？

多态是指父类引用指向子类对象，运行时根据实际类型调用对应方法。底层通过虚方法表（vtable）实现：每个类有一张虚方法表，存放各方法的实际入口地址，调用时查表找到真实方法。编译看左边（类型检查），运行看右边（实际执行）。

### 8. 枚举的本质是什么？

枚举是 `final class` 继承自 `java.lang.Enum`，每个枚举值是 `public static final` 的实例。天然单例、线程安全、可序列化。枚举可以有构造方法、字段和方法，但构造方法必须是 private 的。枚举实现单例是最推荐的方式。

### 9. Integer 缓存池的范围？怎么调整？

`Integer.valueOf()` 对 -128~127 范围内的值使用缓存（`IntegerCache`），超出范围创建新对象。可通过 JVM 参数 `-XX:AutoBoxCacheMax=N` 调整上限。`new Integer()` 不使用缓存，总是创建新对象，所以比较时应使用 `equals()` 而非 `==`。

### 10. 内部类有哪些？静态内部类和非静态内部类的区别？

四种内部类：成员内部类、静态内部类、局部内部类、匿名内部类。非静态内部类隐式持有外部类引用，可以访问外部类所有成员（包括 private），但可能导致内存泄漏；静态内部类不持有外部类引用，只能访问外部类的静态成员，推荐优先使用静态内部类。
