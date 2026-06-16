---
title: Java IO 与 NIO 深入解析
date: 2026-06-14 18:00:00
tags:
  - Java
  - IO
  - NIO
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 5.1 字节流与字符流

### 5.1.1 IO 流体系

```
字节流（处理所有二进制数据）
├── InputStream（输入）
│   ├── FileInputStream
│   ├── ByteArrayInputStream
│   ├── BufferedInputStream
│   ├── DataInputStream
│   └── ObjectInputStream
└── OutputStream（输出）
    ├── FileOutputStream
    ├── BufferedOutputStream
    ├── DataOutputStream
    └── ObjectOutputStream

字符流（处理文本数据，自动编解码）
├── Reader（输入）
│   ├── FileReader
│   ├── BufferedReader
│   ├── InputStreamReader（字节→字符的桥梁）
│   └── StringReader
└── Writer（输出）
    ├── FileWriter
    ├── BufferedWriter
    ├── OutputStreamWriter（字符→字节的桥梁）
    └── PrintWriter
```

### 5.1.2 字节流 vs 字符流

| 特性 | 字节流 | 字符流 |
|------|--------|--------|
| 处理单位 | 1 字节 | 1 字符（可能多字节） |
| 适用场景 | 二进制文件（图片、视频、音频） | 文本文件（txt、csv、json） |
| 编码处理 | 无 | 自动处理字符编码 |
| 基类 | InputStream / OutputStream | Reader / Writer |

```java
// 字节流读写
try (FileInputStream fis = new FileInputStream("input.txt");
     FileOutputStream fos = new FileOutputStream("output.txt")) {
    byte[] buffer = new byte[1024];
    int len;
    while ((len = fis.read(buffer)) != -1) {
        fos.write(buffer, 0, len);
    }
}

// 字符流读写
try (BufferedReader br = new BufferedReader(new FileReader("input.txt"));
     BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        bw.write(line);
        bw.newLine();
    }
}
```

---

## 5.2 缓冲流与装饰器模式

### 5.2.1 装饰器模式在 IO 中的应用

IO 流的设计是经典的装饰器模式：在基础功能上逐层添加增强功能。

```java
// 层层装饰：FileInputStream → BufferedInputStream → DataInputStream
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(
        new FileInputStream("data.bin")));

// 读取各种基本类型
int id = dis.readInt();
String name = dis.readUTF();
double price = dis.readDouble();
```

**为什么需要缓冲流？**

```
无缓冲：每次 read/write 都触发系统调用（用户态→内核态），开销大
有缓冲：先读到缓冲区（默认 8192 字节），批量处理，减少系统调用次数
性能提升：通常 10~100 倍
```

### 5.2.2 常用装饰流

```java
// BufferedReader：带缓冲的字符流，提供 readLine()
BufferedReader br = new BufferedReader(new FileReader("test.txt"));

// PrintWriter：方便的打印输出，支持 println、printf
PrintWriter pw = new PrintWriter(new FileWriter("out.txt"));
pw.println("Hello");
pw.printf("Name: %s, Age: %d", "Tom", 25);

// DataInputStream/DataOutputStream：读写基本类型
DataOutputStream dos = new DataOutputStream(new FileOutputStream("data.bin"));
dos.writeInt(100);
dos.writeDouble(3.14);
dos.writeUTF("hello");
```

---

## 5.3 文件操作与序列化

### 5.3.1 序列化与反序列化

```java
// 对象必须实现 Serializable 接口
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // 版本控制

    private String name;
    private transient String password;  // transient 字段不序列化

    // static 字段不序列化（属于类而非对象）
}

// 序列化
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.dat"))) {
    oos.writeObject(new User("Tom", "123"));
}

// 反序列化
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.dat"))) {
    User user = (User) ois.readObject();
}
```

**serialVersionUID 的作用：**
- 序列化时写入数据流，反序列化时用于验证版本一致性
- 如果不显式声明，JVM 会根据类的结构自动生成一个，修改类后反序列化可能失败
- 建议显式声明 `private static final long serialVersionUID = 1L;`

> **面试考点：serialVersionUID 不一致会怎样？**
> 反序列化时抛出 `InvalidClassException`。因为 JVM 比较序列化数据中的 serialVersionUID 与当前类的 serialVersionUID，不一致说明类结构已变更，为避免数据错误直接拒绝反序列化。

---

## 5.4 NIO 核心：Buffer、Channel、Selector

### 5.4.1 传统 IO vs NIO

| 特性 | 传统 IO | NIO |
|------|---------|-----|
| 模型 | 面向流（Stream） | 面向缓冲区（Buffer） |
| 阻塞 | 阻塞 IO | 非阻塞 IO |
| 多路复用 | 不支持 | Selector 支持 |
| 线程模型 | 一连接一线程 | 一线程管理多连接 |

### 5.4.2 Buffer（缓冲区）

```java
// 核心属性
// capacity：容量，不可变
// position：当前读写位置
// limit：读写界限
// mark：标记位置

// 创建 Buffer
ByteBuffer buffer = ByteBuffer.allocate(1024);  // 堆内存
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);  // 堆外内存（零拷贝）

// 写入数据
buffer.put("hello".getBytes());

// 切换到读模式：flip() 将 limit 设为 position，position 设为 0
buffer.flip();

// 读取数据
while (buffer.hasRemaining()) {
    System.out.print((char) buffer.get());
}

// 重置为写模式：clear() 将 position=0, limit=capacity（数据未擦除，后续写入会覆盖）
buffer.clear();

// compact()：将未读数据移到头部，position 设到未读数据之后
buffer.compact();
```

```
写模式：
+---------+-------+
| data... | empty |    position = 已写入数据末尾
+---------+-------+    limit = capacity

flip() 后（读模式）：
+---------+
| data... |              position = 0
+---------+              limit = 原 position
```

### 5.4.3 Channel（通道）

```java
// 文件读取
FileChannel channel = FileChannel.open(Paths.get("test.txt"), StandardOpenOption.READ);
ByteBuffer buffer = ByteBuffer.allocate(1024);
int bytesRead = channel.read(buffer);  // 数据从 channel 读到 buffer

// 文件写入
FileChannel outChannel = FileChannel.open(Paths.get("out.txt"),
    StandardOpenOption.WRITE, StandardOpenOption.CREATE);
buffer.flip();
outChannel.write(buffer);  // 数据从 buffer 写到 channel

// 文件复制（零拷贝）
FileChannel src = FileChannel.open(Paths.get("src.txt"), StandardOpenOption.READ);
FileChannel dst = FileChannel.open(Paths.get("dst.txt"),
    StandardOpenOption.WRITE, StandardOpenOption.CREATE);
src.transferTo(0, src.size(), dst);  // 直接在内核态传输，不经过用户态
```

### 5.4.4 Selector（多路复用器）

```java
// 创建 Selector
Selector selector = Selector.open();

// 将 Channel 注册到 Selector，关注事件
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);  // 必须是非阻塞模式
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // 阻塞，直到有事件就绪

    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> iter = selectedKeys.iterator();

    while (iter.hasNext()) {
        SelectionKey key = iter.next();

        if (key.isAcceptable()) {
            // 新连接
            ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
            SocketChannel clientChannel = ssc.accept();
            clientChannel.configureBlocking(false);
            clientChannel.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            // 可读数据
            SocketChannel clientChannel = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            clientChannel.read(buffer);
        }

        iter.remove();  // 必须手动移除，否则下次还会处理
    }
}
```

**四种事件类型：**
- `OP_ACCEPT`：服务端接受连接
- `OP_CONNECT`：客户端连接就绪
- `OP_READ`：可读
- `OP_WRITE`：可写

---

## 5.5 NIO.2 与 Path/Files API

```java
// Path：文件路径
Path path = Paths.get("/home", "user", "test.txt");
Path path = Path.of("/home/user/test.txt");  // JDK 11+

// Files：文件操作工具类
Files.createDirectories(Paths.get("/home/user/newDir"));
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
Files.move(src, dst);
Files.deleteIfExists(path);
Files.readString(path);      // JDK 11+，读取全部内容为 String
Files.writeString(path, "content");  // JDK 11+
List<String> lines = Files.readAllLines(path);
byte[] bytes = Files.readAllBytes(path);
```

---

## 5.6 文件监控服务（WatchService）

```java
WatchService watchService = FileSystems.getDefault().newWatchService();
Path dir = Paths.get("/home/user/watch");
dir.register(watchService,
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_MODIFY,
    StandardWatchEventKinds.ENTRY_DELETE);

while (true) {
    WatchKey key = watchService.take();  // 阻塞
    for (WatchEvent<?> event : key.pollEvents()) {
        System.out.println(event.kind() + ": " + event.context());
    }
    key.reset();
}
```

---

## 面试精选

### 1. IO 和 NIO 的区别？

IO 面向流，阻塞式，一个连接一个线程；NIO 面向缓冲区，非阻塞式，通过 Selector 实现多路复用，一个线程管理多个连接。NIO 适合高并发、连接数多但每个连接数据量不大的场景（如聊天服务器）；传统 IO 适合连接少、数据量大的场景（如文件传输）。

### 2. NIO 的三大核心组件？

Buffer：数据容器，核心属性有 capacity、position、limit。Channel：双向数据通道（FileChannel、SocketChannel 等）。Selector：多路复用器，一个线程通过 Selector 监控多个 Channel 的 IO 事件（ACCEPT、CONNECT、READ、WRITE）。

### 3. 什么是零拷贝？Java 中怎么实现？

传统 IO：数据经历 磁盘→内核缓冲区→用户缓冲区→内核缓冲区→网卡，4 次拷贝 4 次上下文切换。零拷贝：数据直接在内核态传输，减少用户态和内核态之间的拷贝。Java 中 `FileChannel.transferTo()` 底层使用操作系统的 sendfile 系统调用实现零拷贝。MappedByteBuffer 也是零拷贝的一种方式。

### 4. ByteBuffer 的 flip、clear、compact 的区别？

flip()：切换写模式到读模式，limit = position，position = 0。clear()：重置为写模式，position = 0，limit = capacity，数据不擦除。compact()：保留未读数据，将已读数据丢弃，position 移到未读数据末尾，继续写模式。

### 5. 什么是序列化？serialVersionUID 的作用？

序列化是将对象转换为字节流的过程，反序列化是逆过程。对象必须实现 Serializable 接口。serialVersionUID 用于版本控制，反序列化时比较版本号是否一致。不一致会抛 InvalidClassException。static 和 transient 字段不参与序列化。

### 6. 装饰器模式在 IO 中的应用？

IO 流的设计就是装饰器模式。基础流（如 FileInputStream）提供基本的字节读写，装饰流（如 BufferedInputStream、DataInputStream）在不修改基础流的前提下增加缓冲、读写基本类型等增强功能。可以层层嵌套，灵活组合。
