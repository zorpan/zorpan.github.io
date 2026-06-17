---
title: Redis 深度解析：数据结构、持久化与分布式锁
date: 2026-06-16 14:00:00
tags:
  - Redis
  - 缓存
  - 分布式锁
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 12.1 数据类型及底层数据结构

### 12.1.1 五种基本数据类型

| 类型 | 底层编码 | 说明 |
|------|---------|------|
| String | int / embstr / raw (SDS) | 最常用，可存字符串、数字、二进制 |
| Hash | ziplist / hashtable | 键值对，适合存对象 |
| List | quicklist（ziplist + 双向链表） | 有序列表，支持头尾操作 |
| Set | intset / hashtable | 无序不重复集合 |
| ZSet | ziplist / skiplist + hashtable | 有序集合，每个元素有分数 |

### 12.1.2 核心底层数据结构

**SDS（Simple Dynamic String）**

```c
struct sdshdr {
    int len;      // 已使用长度
    int alloc;    // 分配的总长度
    char buf[];   // 字节数组
};
```

- 二进制安全：可存储任意二进制数据
- O(1) 获取长度（不需要遍历）
- 预分配和惰性释放减少内存分配次数
- 兼容 C 字符串函数

**Dict（哈希表）**

```
dict
├── ht[0]：主哈希表
└── ht[1]：渐进式 rehash 时使用

每个哈希表：
├── table[]：桶数组
├── size：桶数量
├── used：已存键值对数量
└── sizemask：size - 1（用于取模）

哈希冲突：链地址法
扩容：used / size > 1（无 BGSAVE 时）或 > 5（有 BGSAVE 时）
收缩：used / size < 0.1
渐进式 rehash：每次操作迁移一个桶，避免一次性迁移阻塞
```

**SkipList（跳表）**

```
Level 4:  1 ───────────────────────────→ 9 → NULL
Level 3:  1 ──────→ 5 ──────────────────→ 9 → NULL
Level 2:  1 ──→ 3 → 5 ──→ 7 ────────────→ 9 → NULL
Level 1:  1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → NULL
```

- 多层链表结构，查找时间复杂度 O(log n)
- ZSet 的底层实现（当元素较多时）
- 每个节点随机分配层高（概率 p=0.25）
- 比红黑树实现更简单，范围查询更高效

**ZSet 为什么用跳表不用红黑树？**
1. 跳表实现更简单，代码更易维护
2. 范围查询更高效（跳表叶子节点链表直接遍历）
3. 并发友好（加锁粒度更细）
4. 内存占用可灵活调整（通过调整层高概率）

**QuickList（快速列表，List 的底层）**

```
quicklist = 双向链表 + 每个节点是一个 ziplist
兼顾了链表的灵活操作和 ziplist 的内存紧凑
```

---

## 12.2 持久化机制

### 12.2.1 RDB（Redis Database）

```
将某个时间点的全量数据生成快照，保存为 dump.rdb 文件

触发方式：
- save：阻塞主线程（生产禁用）
- bgsave：fork 子进程，利用 COW（Copy-On-Write）机制
- 配置自动触发：save 3600 1（3600 秒内有 1 次修改）

优势：文件紧凑，恢复速度快
劣势：可能丢失最后一次快照后的数据
```

**COW（写时复制）机制：**
1. bgsave fork 子进程，共享父进程内存页
2. 父进程有写操作时，复制一份内存页再修改（COW）
3. 子进程始终看到 fork 时的内存快照

### 12.2.2 AOF（Append Only File）

```
记录每一条写命令，以追加方式写入日志文件

写入策略：
- always：每次写命令都 fsync（最安全，性能最差）
- everysec：每秒 fsync（推荐，最多丢 1 秒数据）
- no：由操作系统决定何时 fsync（性能最好，可能丢数据）

AOF 重写：
- 合并冗余命令，压缩 AOF 文件
- bgrewriteaof 触发，fork 子进程重写
- 重写期间新命令写入 AOF 缓冲区 + AOF 重写缓冲区
- 子进程重写完成后，主进程将重写缓冲区内容追加到新 AOF 文件
```

### 12.2.3 混合持久化（Redis 4.0+）

```
AOF 重写时：前半部分用 RDB 格式（快），后半部分用 AOF 格式（增量）
兼顾 RDB 的快速恢复和 AOF 的低数据丢失
```

| 方式 | 数据安全 | 恢复速度 | 文件大小 |
|------|---------|---------|---------|
| RDB | 可能丢数据 | 快 | 小 |
| AOF(everysec) | 最多丢 1 秒 | 慢 | 大 |
| 混合持久化 | 最多丢 1 秒 | 较快 | 中 |

---

## 12.3 内存淘汰策略

### 12.3.1 八种淘汰策略

| 策略 | 范围 | 算法 | 说明 |
|------|------|------|------|
| noeviction | — | — | 不淘汰，内存满时拒绝写入（默认） |
| allkeys-lru | 所有 key | LRU | 最近最少使用 |
| allkeys-lfu | 所有 key | LFU | 最不经常使用（Redis 4.0+） |
| allkeys-random | 所有 key | 随机 | 随机淘汰 |
| volatile-lru | 设置了过期时间的 key | LRU | 最近最少使用 |
| volatile-lfu | 设置了过期时间的 key | LFU | 最不经常使用 |
| volatile-random | 设置了过期时间的 key | 随机 | 随机淘汰 |
| volatile-ttl | 设置了过期时间的 key | TTL | 淘汰最快过期的 |

**推荐：allkeys-lfu（热点数据保留）或 allkeys-lru（通用场景）**

### 12.3.2 LRU vs LFU

- **LRU（Least Recently Used）**：淘汰最久未访问的数据。问题：偶尔访问一次的冷数据不会被淘汰
- **LFU（Least Frequently Used）**：淘汰访问频率最低的数据。更精准地识别热点数据
- Redis 的 LRU/LFU 是近似实现（采样法），不是精确实现（精确实现需要额外链表，内存开销大）

---

## 12.4 Redis 单线程模型与多线程演进

### 12.4.1 为什么单线程还这么快？

```
Redis 单线程模型：
1. 基于 epoll（Linux）的 IO 多路复用
2. 纯内存操作（数据在内存中，无磁盘 IO）
3. 单线程避免了上下文切换和锁竞争
4. 高效数据结构（SDS、跳表、哈希表等）
```

### 12.4.2 多线程演进

- Redis 4.0：引入多线程处理异步任务（删除大 key、AOF 重写）
- Redis 6.0：引入多线程 IO（网络 IO 用多线程，命令执行仍单线程）
- 核心命令执行仍然是单线程，保证线程安全

---

## 12.5 主从复制、哨兵机制、Cluster 集群

### 12.5.1 主从复制

```
全量复制（首次连接）：
1. Slave 发送 PSYNC ? -1
2. Master 执行 BGSAVE 生成 RDB，发送给 Slave
3. Slave 加载 RDB
4. Master 将期间的写命令发送给 Slave

增量复制（断线重连）：
1. Slave 携带 offset 重连
2. Master 检查 repl_backlog 中是否有 Slave 需要的数据
3. 有则增量发送，无则全量复制
```

### 12.5.2 哨兵（Sentinel）

```
Sentinel 集群（至少 3 个节点）
├── 监控：检测主从节点是否正常工作
├── 通知：故障时通知客户端
└── 自动故障转移：主节点下线时选举新主节点

故障判定：
- 主观下线（SDOWN）：单个 Sentinel 认为节点下线
- 客观下线（ODOWN）：quorum 个 Sentinel 都认为主节点下线
```

### 12.5.3 Cluster 集群

```
Redis Cluster：
- 数据分片：16384 个槽（slot），每个节点负责一部分槽
- 去中心化：任意节点都可以接收请求，不在本节点的槽则重定向
- Gossip 协议：节点间互相交换状态信息

槽位计算：CRC16(key) % 16384

扩缩容：
- 添加节点：从其他节点迁移部分槽
- 移除节点：将该节点的槽迁移到其他节点
```

---

## 12.6 分布式锁

### 12.6.1 基于 SETNX

```bash
# 基本加锁（原子操作）
SET lock_key unique_value NX EX 30
# NX：不存在才设置
# EX 30：过期时间 30 秒

# 释放锁（Lua 脚本保证原子性）
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### 12.6.2 Redisson 实现（推荐）

```java
RLock lock = redisson.getLock("myLock");
try {
    lock.lock(30, TimeUnit.SECONDS);  // 自动续期
    // 业务逻辑
} finally {
    lock.unlock();
}
```

**Redisson 看门狗机制：**
- 加锁时默认过期时间 30 秒
- 后台线程每 10 秒检查锁是否还在，自动续期到 30 秒
- 业务完成后手动释放锁，不再续期

### 12.6.3 RedLock（有争议）

```
向 N 个独立的 Redis 节点（推荐 5 个）加锁
多数（N/2 + 1）加锁成功才算成功
总耗时小于锁的有效期才算有效

问题：时钟漂移、网络分区可能导致不安全
Martin Kleppmann 和 antirez 的著名争论
实际生产中大多数场景用 Redisson 单节点即可
```

---

## 12.7 缓存穿透、击穿、雪崩

### 12.7.1 缓存穿透

```
查询一个一定不存在的数据，缓存和数据库都没有
每次请求都打到数据库

解决方案：
1. 缓存空值：数据库查不到也缓存 null，设置较短过期时间
2. 布隆过滤器：在缓存前加一层布隆过滤器
   - 将所有可能存在的数据哈希到位数组中
   - 请求先查布隆过滤器，不存在则直接返回
   - 可能误判存在，不会误判不存在
```

### 12.7.2 缓存击穿

```
某个热点 key 过期的瞬间，大量请求打到数据库

解决方案：
1. 互斥锁：只允许一个线程重建缓存，其他线程等待
2. 热点 key 永不过期：后台异步更新
3. 逻辑过期：缓存中存过期时间，过期后由一个线程去更新
```

### 12.7.3 缓存雪崩

```
大量 key 同时过期，或 Redis 宕机，大量请求打到数据库

解决方案：
1. 过期时间加随机值：避免同时过期
2. 多级缓存：L1 本地缓存（Caffeine） + L2 Redis
3. 熔断降级：数据库压力大时返回默认值
4. Redis 高可用：哨兵 / Cluster
```

| 问题 | 本质 | 核心方案 |
|------|------|---------|
| 穿透 | 查询不存在的数据 | 布隆过滤器 + 缓存空值 |
| 击穿 | 热点 key 过期 | 互斥锁 + 永不过期 |
| 雪崩 | 大量 key 同时失效 | 过期加随机值 + 多级缓存 |

---

## 12.8 延迟队列与 Stream

### 12.8.1 延迟队列实现

```bash
# 方案1：ZSET 实现延迟队列
ZADD delay_queue <timestamp> <message>
# 消费者轮询
ZRANGEBYSCORE delay_queue 0 <current_timestamp> LIMIT 0 10
ZREM delay_queue <message>

# 方案2：Redis 5.0+ Stream（推荐）
XADD mystream * field1 value1
XREAD COUNT 10 BLOCK 5000 STREAMS mystream 0

# 消费者组
XGROUP CREATE mystream mygroup 0
XREADGROUP GROUP mygroup consumer1 COUNT 1 BLOCK 5000 STREAMS mystream >
XACK mystream mygroup <message_id>
```

---

## 12.9 Lua 脚本与事务

### 12.9.1 Redis 事务

```bash
MULTI
SET key1 value1
SET key2 value2
EXEC
# 事务不支持回滚！EXEC 后某条命令失败，其他命令仍会执行
```

### 12.9.2 Lua 脚本（推荐）

```bash
# 原子执行多条命令
EVAL "redis.call('set', KEYS[1], ARGV[1]) redis.call('expire', KEYS[1], ARGV[2])" 1 mykey myvalue 30

# 优势：
# 1. 原子性：Lua 脚本作为一个整体执行
# 2. 减少网络开销：多条命令一次发送
# 3. 可复用：SCRIPT LOAD 缓存脚本
```

---

## 面试精选

### 1. Redis 有哪些数据类型？底层数据结构是什么？

五种基本类型：String（SDS）、Hash（ziplist/hashtable）、List（quicklist）、Set（intset/hashtable）、ZSet（ziplist/skiplist+hashtable）。核心数据结构：SDS（动态字符串）、Dict（哈希表）、SkipList（跳表）、QuickList（快速列表）、IntSet（整数集合）。当元素较少时使用压缩结构（ziplist/intset）节省内存。

### 2. Redis 为什么用跳表不用红黑树实现 ZSet？

1）跳表实现更简单，代码更易维护；2）范围查询更高效，跳表叶子节点链表直接遍历；3）并发友好，加锁粒度更细；4）内存占用可灵活调整。红黑树虽然查找也是 O(log n)，但范围查询需要中序遍历，不如跳表方便。

### 3. RDB 和 AOF 的区别？怎么选？

RDB：全量快照，恢复快，文件小，但可能丢数据。AOF：记录每条命令，最多丢 1 秒（everysec），文件大，恢复慢。推荐混合持久化（Redis 4.0+）：AOF 重写时前半部分用 RDB 格式，后半部分用 AOF 格式，兼顾恢复速度和数据安全。

### 4. Redis 的持久化流程是怎样的？

RDB：bgsave fork 子进程，利用 COW 机制生成快照，不阻塞主进程。AOF：写命令先追加到 AOF 缓冲区，根据策略（always/everysec/no）fsync 到磁盘。AOF 重写时 fork 子进程，期间新命令同时写入 AOF 缓冲区和重写缓冲区，重写完成后替换旧 AOF 文件。

### 5. 缓存穿透、击穿、雪崩的区别和解决方案？

穿透：查询不存在的数据，每次打到数据库。方案：布隆过滤器 + 缓存空值。击穿：热点 key 过期瞬间大量请求打到数据库。方案：互斥锁重建缓存 + 热点 key 永不过期。雪崩：大量 key 同时过期或 Redis 宕机。方案：过期时间加随机值 + 多级缓存 + 熔断降级。

### 6. Redis 分布式锁怎么实现？有什么问题？

基本实现：SET key value NX EX 30（原子操作），释放时用 Lua 脚本比较 value 再删除。Redisson：看门狗机制自动续期（默认 30 秒，每 10 秒续一次），业务完成后手动释放。问题：主从模式下主节点宕机可能导致锁丢失（可用 RedLock 缓解，但有争议）。

### 7. Redis 单线程为什么还这么快？

1）纯内存操作，无磁盘 IO；2）IO 多路复用（epoll），单线程处理大量连接；3）单线程避免上下文切换和锁竞争；4）高效数据结构（SDS、跳表、哈希表）。Redis 6.0 引入多线程 IO 处理网络收发，但命令执行仍单线程。

### 8. Redis Cluster 的原理？

数据分片：16384 个槽，每个节点负责一部分，CRC16(key) % 16384 定位槽。去中心化：任意节点接收请求，不在本节点则 MOVED 重定向。节点间通过 Gossip 协议交换状态。扩缩容通过迁移槽实现。每个主节点可以有从节点做故障转移。

### 9. Redis 和 Memcached 的区别？

1）Redis 支持多种数据结构（String/List/Hash/Set/ZSet），Memcached 只支持 String；2）Redis 支持持久化（RDB/AOF），Memcached 不支持；3）Redis 支持集群和哨兵，Memcached 需要客户端分片；4）Redis 单线程（6.0 多线程 IO），Memcached 多线程；5）Redis 支持 Lua 脚本和事务。
