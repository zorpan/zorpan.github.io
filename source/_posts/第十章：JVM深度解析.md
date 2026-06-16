---
title: JVM 深度解析：内存模型、GC 与调优实战
date: 2026-06-15 16:00:00
tags:
  - Java
  - JVM
  - 垃圾回收
  - 性能调优
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 10.1 JVM 内存模型

### 10.1.1 运行时数据区

```
┌─────────────────────────────────────────────┐
│                   JVM 进程                   │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │           线程私有（每个线程一份）        │ │
│  │                                        │ │
│  │  ┌──────────────┐  ┌──────────────┐   │ │
│  │  │ 程序计数器    │  │ 虚拟机栈      │   │ │
│  │  │ (PC Register) │  │ (VM Stack)   │   │ │
│  │  └──────────────┘  └──────────────┘   │ │
│  │  ┌──────────────┐                     │ │
│  │  │ 本地方法栈    │                     │ │
│  │  │ (Native Stack)│                     │ │
│  │  └──────────────┘                     │ │
│  └────────────────────────────────────────┘ │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │           线程共享                       │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐  │ │
│  │  │           堆 (Heap)              │  │ │
│  │  │  ┌─────────┐  ┌──────────────┐  │  │ │
│  │  │  │ 新生代   │  │   老年代      │  │  │ │
│  │  │  │Eden│S0│S1│  │  Old Gen     │  │  │ │
│  │  │  └─────────┘  └──────────────┘  │  │ │
│  │  └──────────────────────────────────┘  │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐  │ │
│  │  │   方法区 / 元空间 (Metaspace)     │  │ │
│  │  │   (类信息、常量池、静态变量)       │  │ │
│  │  └──────────────────────────────────┘  │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 10.1.2 各区域详解

**程序计数器（PC Register）**
- 线程私有，当前执行的字节码指令的地址
- 唯一不会 OOM 的区域
- 如果执行的是 native 方法，计数器值为空（Undefined）

**虚拟机栈（VM Stack）**
- 线程私有，每个方法调用创建一个栈帧
- 栈帧包含：局部变量表、操作数栈、动态链接、方法返回地址
- 栈深度超限 → StackOverflowError（如递归太深）
- 栈动态扩展失败 → OutOfMemoryError

**本地方法栈（Native Method Stack）**
- 与虚拟机栈类似，但为 native 方法服务
- HotSpot 中本地方法栈和虚拟机栈合二为一

**堆（Heap）**
- 线程共享，对象实例和数组都在堆上分配
- GC 管理的主要区域
- 物理上可以不连续，逻辑上连续

**方法区 / 元空间（Metaspace）**
- JDK 7 及之前：永久代（PermGen），使用 JVM 堆内存，有 `PermSize` 限制
- JDK 8+：元空间（Metaspace），使用本地内存（OS 内存），默认不限大小
- 存储：类信息、常量池、静态变量、即时编译器编译后的代码

> **面试考点：JDK 8 为什么用元空间替代永久代？**
> 1. 永久代大小固定，容易 OOM（`PermGen space`）
> 2. 字符串常量池在 JDK 7 已移到堆中，永久代意义不大
> 3. 元空间使用本地内存，默认可以动态扩展，减少 OOM 风险
> 4. 方便 HotSpot 与 JRockit 代码合并（JRockit 没有永久代）

---

## 10.2 垃圾回收算法

### 10.2.1 对象存活判定

**引用计数法：**
- 每个对象维护一个引用计数器
- 引用 +1，失效 -1
- 缺点：无法解决循环引用
- JVM 不使用此方案

**可达性分析（GC Roots）：**
- 从 GC Roots 出发，沿引用链遍历，不可达的对象即为垃圾
- GC Roots 包括：
  - 虚拟机栈中引用的对象（局部变量）
  - 方法区中静态变量引用的对象
  - 方法区中常量引用的对象
  - 本地方法栈中 JNI 引用的对象
  - synchronized 锁持有的对象

### 10.2.2 四种引用类型

| 引用类型 | GC 时回收 | 用途 |
|---------|----------|------|
| 强引用 `Object o = new Object()` | 不回收 | 普通使用 |
| 软引用 `SoftReference` | 内存不足时回收 | 缓存（如图片缓存） |
| 弱引用 `WeakReference` | 下次 GC 时回收 | ThreadLocalMap 的 key |
| 虚引用 `PhantomReference` | 随时回收 | 跟踪对象被 GC 的状态 |

### 10.2.3 垃圾回收算法

**标记-清除（Mark-Sweep）**
- 标记可回收对象，然后清除
- 缺点：产生内存碎片
- 适用：老年代（对象存活率高）

**标记-复制（Copying）**
- 将存活对象复制到另一块内存，然后清空原区域
- 优点：无碎片
- 缺点：空间浪费（可用内存减半）
- 适用：新生代（Eden:S0:S1 = 8:1:1，实际可用 90%）

**标记-整理（Mark-Compact）**
- 标记存活对象，然后向一端移动，清理边界外内存
- 优点：无碎片
- 缺点：移动对象开销大
- 适用：老年代

**分代收集**
- 新生代（Young Gen）：对象存活率低，用复制算法
- 老年代（Old Gen）：对象存活率高，用标记-清除或标记-整理

---

## 10.3 垃圾收集器

### 10.3.1 收集器全景图

```
新生代                    老年代
Serial ──────────────→ Serial Old
ParNew ──────────────→ CMS / Serial Old
Parallel Scavenge ───→ Parallel Old / Serial Old

G1（整堆收集）
ZGC（低延迟）
Shenandoah（低延迟）
```

### 10.3.2 Serial / Serial Old

- 单线程，STW（Stop-The-World）
- 简单高效，适合客户端程序
- 新生代用复制算法，老年代用标记-整理

### 10.3.3 Parallel Scavenge / Parallel Old（JDK 8 默认）

- 多线程并行收集，关注吞吐量
- 吞吐量 = 运行用户代码时间 / (运行用户代码时间 + GC 时间)
- 适合后台计算密集型任务
- `-XX:MaxGCPauseMillis` 控制最大停顿时间
- `-XX:GCTimeRatio` 控制吞吐量

### 10.3.4 CMS（Concurrent Mark Sweep）

- 关注低停顿，老年代收集器
- 四个阶段：
  1. **初始标记**（STW）：标记 GC Roots 直接关联的对象，速度快
  2. **并发标记**：从 GC Roots 出发遍历整个引用链，与用户线程并发
  3. **重新标记**（STW）：修正并发标记期间变动的引用
  4. **并发清除**：清除不可达对象，与用户线程并发
- 缺点：CPU 敏感、浮动垃圾、内存碎片（标记-清除）
- JDK 14 被移除

### 10.3.5 G1（Garbage-First）（JDK 9+ 默认）

- 整堆收集器，兼顾吞吐量和低停顿
- 将堆划分为多个等大的 Region（1~32MB）
- 每个 Region 可以是 Eden、Survivor、Old 或 Humongous（大对象）
- 四个阶段：
  1. **初始标记**（STW）：标记 GC Roots 直接关联的对象
  2. **并发标记**：遍历引用链
  3. **最终标记**（STW）：处理并发标记遗留的 SATB 记录
  4. **筛选回收**（STW）：对每个 Region 的回收价值排序，优先回收价值最大的 Region
- 可预测的停顿时间：`-XX:MaxGCPauseMillis=200`（默认 200ms）
- 适用：大堆（6GB+）、低延迟要求

### 10.3.6 ZGC（JDK 11+，JDK 15 正式）

- 超低延迟（停顿时间 < 1ms，不随堆大小增长）
- 使用染色指针（Colored Pointer）和读屏障（Load Barrier）
- 支持 TB 级堆
- 不分代（JDK 21 引入分代 ZGC）
- 适合超大堆、对延迟极其敏感的场景

| 收集器 | 区域 | 算法 | 特点 | 适用场景 |
|--------|------|------|------|---------|
| Serial | 新生代 | 复制 | 单线程 STW | 客户端 |
| Parallel | 新生代 | 复制 | 多线程吞吐优先 | 后台计算 |
| CMS | 老年代 | 标记-清除 | 低停顿（已废弃） | 互联网应用 |
| G1 | 整堆 | 分 Region | 兼顾吞吐和延迟 | 大堆通用 |
| ZGC | 整堆 | 染色指针 | 超低延迟 | 超大堆 |

---

## 10.4 类加载机制与双亲委派模型

### 10.4.1 类的生命周期

```
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
  └────── 连接 ──────┘
```

- **加载**：通过类的全限定名获取字节流 → 转化为方法区的运行时数据结构 → 生成 Class 对象
- **验证**：文件格式、元数据、字节码、符号引用验证
- **准备**：为类的静态变量分配内存并设置零值（`static int a = 10` 此时 a = 0，不是 10）
- **解析**：将符号引用替换为直接引用
- **初始化**：执行 `<clinit>()` 方法（static 块和 static 变量赋值）

### 10.4.2 双亲委派模型

```
              ┌──────────────────┐
              │ Bootstrap ClassLoader │  ← 加载 rt.jar（String、Object等）
              │ （C++ 实现，无 Java 对象）│
              └────────┬─────────┘
                       │ 委派
              ┌────────▼─────────┐
              │ Extension ClassLoader │  ← 加载 ext 目录（javax.*）
              └────────┬─────────┘
                       │ 委派
              ┌────────▼─────────┐
              │ Application ClassLoader │  ← 加载 classpath（用户代码）
              └────────┬─────────┘
                       │ 委派
              ┌────────▼─────────┐
              │ Custom ClassLoader │  ← 自定义加载器
              └──────────────────┘
```

**工作流程：**
1. 子加载器先检查类是否已加载
2. 未加载则委托给父加载器
3. 父加载器无法加载时，子加载器才自己加载

**为什么要双亲委派？**
- **安全性**：防止用户自定义 `java.lang.String` 替换核心类
- **唯一性**：保证同一个类只被加载一次

### 10.4.3 打破双亲委派

```java
// 场景1：JNDI（线程上下文类加载器）
// 父加载器需要加载子加载器路径下的类（如 JDBC 驱动）
Thread.currentThread().getContextClassLoader();

// 场景2：Tomcat（每个 Web 应用独立的类加载器）
// Common → Catalina → Shared → WebApp ClassLoader
// WebApp ClassLoader 优先自己加载，打破了"先委派父加载器"的规则

// 场景3：OSGi（网状类加载）
// 模块之间互相引用，不再是树形结构
```

---

## 10.5 自定义类加载器

```java
public class MyClassLoader extends ClassLoader {
    private final String classPath;

    public MyClassLoader(String classPath) {
        this.classPath = classPath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] data = loadByte(name);
            return defineClass(name, data, 0, data.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }

    private byte[] loadByte(String name) throws IOException {
        String path = classPath + File.separatorChar +
            name.replace('.', File.separatorChar) + ".class";
        return Files.readAllBytes(Path.of(path));
    }
}

// 使用
MyClassLoader loader = new MyClassLoader("/tmp/classes");
Class<?> clazz = loader.loadClass("com.example.User");
Object instance = clazz.getDeclaredConstructor().newInstance();
```

---

## 10.6 JIT 编译与逃逸分析

### 10.6.1 JIT 编译

```
Java 源码 → javac → 字节码 → 解释执行（慢）
                                    ↓ 热点代码
                              JIT 编译 → 机器码（快）
```

- **热点代码**：被多次调用的方法或循环体
- **判断标准**：方法调用计数器 + 回边计数器
- **C1 编译器**（Client Compiler）：快速编译，简单优化
- **C2 编译器**（Server Compiler）：深度优化，编译慢
- **分层编译**：先 C1 编译，热点代码再用 C2 重新编译（JDK 8+）

### 10.6.2 逃逸分析

分析对象的作用域，判断对象是否逃逸出方法或线程。

```java
// 未逃逸：对象只在方法内部使用
public void method() {
    Point p = new Point(1, 2);  // p 未逃逸出方法
    System.out.println(p.x + p.y);
}
```

**逃逸分析的三种优化：**

**1. 栈上分配（Stack Allocation）**
- 对象未逃逸时，在栈上分配而非堆上
- 方法结束自动回收，无需 GC

**2. 标量替换（Scalar Replacement）**
- 将对象拆解为基本类型变量
- `new Point(1, 2)` → `int x = 1; int y = 2;`

**3. 锁消除（Lock Elimination）**
- 对象未逃逸出线程时，消除同步锁
- `StringBuffer` 在方法内部使用时，锁会被消除

---

## 10.7 JVM 调优实战

### 10.7.1 常用 JVM 参数

```bash
# 堆内存
-Xms512m           # 初始堆大小
-Xmx1024m          # 最大堆大小
-Xmn256m           # 新生代大小
-XX:MetaspaceSize=256m   # 元空间初始大小
-XX:MaxMetaspaceSize=512m # 元空间最大大小

# GC 收集器
-XX:+UseG1GC                # 使用 G1
-XX:MaxGCPauseMillis=200    # G1 最大停顿时间
-XX:+UseZGC                 # 使用 ZGC（JDK 11+）

# GC 日志（JDK 9+）
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# 堆转储
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof
```

### 10.7.2 常用诊断工具

```bash
# jps：查看 Java 进程
jps -l

# jinfo：查看 JVM 参数
jinfo -flags <PID>

# jstack：线程栈快照（排查死锁、线程阻塞）
jstack <PID>

# jmap：堆转储
jmap -dump:format=b,file=heap.hprof <PID>

# jstat：GC 统计
jstat -gcutil <PID> 1000 10  # 每秒打印 GC 统计，共 10 次

# Arthas（阿里开源的 Java 诊断工具）
# 线上排查首选，无需重启 JVM
java -jar arthas-boot.jar
```

---

## 10.8 OOM 问题排查

### 10.8.1 常见 OOM 类型

| OOM 类型 | 原因 | 排查方向 |
|---------|------|---------|
| `Java heap space` | 堆内存不足 | 内存泄漏？堆太小？ |
| `Metaspace` | 类加载过多 | 动态生成类过多？ |
| `GC overhead limit exceeded` | GC 耗时过长 | 内存泄漏 |
| `Direct buffer memory` | 堆外内存不足 | NIO Buffer 未释放？ |
| `Unable to create new native thread` | 线程数超限 | 线程池配置？ |
| `StackOverflowError` | 栈深度超限 | 递归太深？ |

### 10.8.2 排查步骤

```
1. 确认 OOM 类型 → 看异常信息
2. 获取堆转储 → -XX:+HeapDumpOnOutOfMemoryError 或 jmap -dump
3. 分析堆转储 → MAT（Memory Analyzer Tool）或 VisualVM
4. 查找大对象 / 泄漏对象 → MAT 的 Leak Suspects 报告
5. 定位代码 → 查看 GC Roots 引用链
6. 修复 → 修复代码或调整 JVM 参数
```

---

## 10.9 字节码结构与执行引擎

### 10.9.1 class 文件结构

```
ClassFile {
    u4 magic;                    // 魔数：CAFEBABE
    u2 minor_version;            // 次版本号
    u2 major_version;            // 主版本号（52=JDK8, 61=JDK17）
    u2 constant_pool_count;      // 常量池计数
    cp_info constant_pool[];     // 常量池
    u2 access_flags;             // 访问标志（public、abstract 等）
    u2 this_class;               // 当前类
    u2 super_class;              // 父类
    u2 interfaces_count;         // 接口计数
    u2 interfaces[];             // 接口表
    u2 fields_count;             // 字段计数
    field_info fields[];         // 字段表
    u2 methods_count;            // 方法计数
    method_info methods[];       // 方法表
    u2 attributes_count;         // 属性计数
    attribute_info attributes[]; // 属性表
}
```

### 10.9.2 查看字节码

```bash
# 使用 javap 反编译
javap -v -p MyClass.class

# 常见字节码指令
# 加载/存储：iload, aload, istore, astore
# 运算：iadd, isub, imul, idiv
# 比较：if_icmpne, if_icmpeq
# 栈操作：dup, pop, swap
# 方法调用：invokevirtual, invokeinterface, invokespecial, invokestatic, invokedynamic
# 对象创建：new, newarray, anewarray
```

---

## 面试精选

### 1. JVM 内存模型有哪些区域？

线程私有：程序计数器（当前指令地址）、虚拟机栈（栈帧：局部变量表、操作数栈、动态链接、返回地址）、本地方法栈（native 方法）。线程共享：堆（对象实例，GC 主要区域）、方法区/元空间（类信息、常量池、静态变量）。

### 2. 垃圾回收算法有哪些？

标记-清除（碎片多）、标记-复制（空间浪费，无碎片）、标记-整理（无碎片，移动开销大）、分代收集（新生代用复制，老年代用标记-清除/整理）。HotSpot 新生代用 Eden:S0:S1 = 8:1:1 的复制算法。

### 3. G1 和 CMS 的区别？

CMS 关注低停顿，标记-清除，有碎片，四阶段（初始标记→并发标记→重新标记→并发清除），JDK 14 已废弃。G1 关注可预测停顿，分 Region 整堆收集，可设最大停顿时间，JDK 9+ 默认收集器。G1 适合大堆（6GB+），CMS 适合中等堆。

### 4. 什么是双亲委派？为什么要这样设计？

类加载时先委托父加载器加载，父加载器无法加载时才自己加载。目的：1）安全性，防止自定义类替换核心类（如 java.lang.String）；2）唯一性，保证同一个类只被加载一次。打破双亲委派的场景：JNDI（线程上下文类加载器）、Tomcat（WebApp 优先自己加载）、OSGi（网状结构）。

### 5. 什么时候触发 Full GC？

1）老年代空间不足；2）方法区/元空间不足；3）调用 System.gc()（建议但不强制）；4）CMS GC 出现 Concurrent Mode Failure；5）进入老年代的平均大小大于老年代剩余空间。线上应避免频繁 Full GC，说明堆配置不合理或存在内存泄漏。

### 6. 如何排查 OOM？

1）启动时加上 -XX:+HeapDumpOnOutOfMemoryError；2）OOM 发生后用 MAT 分析堆转储文件；3）Leak Suspects 报告自动定位疑似泄漏点；4）查看 GC Roots 引用链找到持有对象的代码；5）结合 jstat 查看 GC 趋势，确认是内存泄漏还是容量不足。

### 7. 什么是逃逸分析？有什么优化？

分析对象是否逃逸出方法或线程。三种优化：1）栈上分配（未逃逸对象在栈上分配，方法结束自动回收）；2）标量替换（对象拆解为基本类型变量）；3）锁消除（未逃逸出线程的对象，消除同步锁）。

### 8. JDK 8 为什么要用元空间替代永久代？

1）永久代大小固定，容易 OOM；2）字符串常量池在 JDK 7 已移到堆；3）元空间使用本地内存，默认可动态扩展；4）方便 HotSpot 与 JRockit 合并。可通过 -XX:MaxMetaspaceSize 限制元空间大小。
