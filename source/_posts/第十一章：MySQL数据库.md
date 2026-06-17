---
title: MySQL 数据库深度解析：索引、事务与锁机制
date: 2026-06-16 10:00:00
tags:
  - MySQL
  - 索引
  - 事务
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 11.1 SQL 语法进阶与执行顺序

### 11.1.1 SQL 执行顺序

```sql
SELECT DISTINCT column1, AGG_FUNC(column2)
FROM table1
    JOIN table2 ON table1.id = table2.id
    JOIN table3 ON table2.id = table3.id
WHERE condition
GROUP BY column1
HAVING group_condition
ORDER BY column1 ASC
LIMIT n OFFSET m;
```

**实际执行顺序：**

```
FROM → JOIN → ON → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

### 11.1.2 常用进阶语法

```sql
-- 窗口函数（MySQL 8.0+）
SELECT
    name,
    score,
    RANK() OVER (ORDER BY score DESC) as ranking,
    ROW_NUMBER() OVER (PARTITION BY class_id ORDER BY score DESC) as class_rank,
    LAG(score, 1) OVER (ORDER BY id) as prev_score
FROM student;

-- CTE 公共表表达式（MySQL 8.0+）
WITH dept_salary AS (
    SELECT dept_id, AVG(salary) as avg_salary
    FROM employee GROUP BY dept_id
)
SELECT e.name, e.salary, d.avg_salary
FROM employee e JOIN dept_salary d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_salary;

-- 子查询优化：EXISTS vs IN
-- EXISTS 适合外表小、内表大（找到匹配就返回）
-- IN 适合外表大、内表小（先查内表结果集再匹配）
SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM customers c WHERE c.id = o.customer_id AND c.vip = 1);
```

---

## 11.2 索引原理

### 11.2.1 B+ 树结构

```
                [10 | 20 | 30]           ← 非叶子节点（只存 key）
               /      |      \
    [1,3,5,7] → [10,12,15,18] → [20,25,28] → [30,35,40]   ← 叶子节点（存 key+data）
    ←────────── 双向链表连接 ──────────→
```

**为什么用 B+ 树而不是 B 树、红黑树、Hash？**

| 数据结构 | 问题 |
|---------|------|
| 二叉树 / 红黑树 | 树高太大，IO 次数多 |
| B 树 | 非叶子节点也存数据，每页能存的 key 更少，树更高 |
| Hash | 不支持范围查询、排序 |
| B+ 树 | 非叶子节点只存 key，每页存更多 key → 树更矮 → IO 更少；叶子节点用双向链表连接 → 支持范围查询 |

### 11.2.2 聚簇索引 vs 非聚簇索引

```
聚簇索引（主键索引）：
叶子节点直接存储完整的行数据
一个表只有一个聚簇索引（就是主键）

非聚簇索引（二级索引）：
叶子节点存储的是主键值（不是完整行数据）
通过二级索引查询时需要回表：先查二级索引拿到主键，再通过主键查聚簇索引拿到完整数据
```

### 11.2.3 联合索引与最左前缀

```sql
-- 联合索引 (a, b, c)
-- 索引按 a 排序，a 相同按 b 排序，b 相同按 c 排序

-- 命中索引
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3
WHERE a = 1 AND c = 3          -- 只用到 a
WHERE a = 1 ORDER BY b         -- a 等值 + b 排序，利用索引有序性

-- 不命中索引
WHERE b = 2                     -- 跳过了 a
WHERE b = 2 AND c = 3           -- 跳过了 a
WHERE a = 1 AND b > 2 AND c = 3 -- c 无法使用索引（范围查询后的列失效）
```

---

## 11.3 索引优化策略

### 11.3.1 覆盖索引

```sql
-- 覆盖索引：查询的字段全部在索引中，无需回表
-- 索引 (name, age)
SELECT name, age FROM user WHERE name = 'Tom';  -- 覆盖索引，Extra: Using index
SELECT * FROM user WHERE name = 'Tom';          -- 需要回表
```

### 11.3.2 索引下推（ICP，MySQL 5.6+）

```sql
-- 索引 (name, age)
SELECT * FROM user WHERE name LIKE '张%' AND age = 25;

-- 无 ICP：先通过 name 查出所有"张%"，回表后再过滤 age
-- 有 ICP：在索引层直接过滤 age，减少回表次数
```

### 11.3.3 索引失效的常见场景

```sql
-- 1. 对索引列做函数/运算
WHERE YEAR(create_time) = 2024    -- 失效
WHERE create_time >= '2024-01-01'  -- 命中

-- 2. 隐式类型转换
WHERE phone = 13800138000          -- phone 是 varchar，传入 int，索引失效
WHERE phone = '13800138000'        -- 命中

-- 3. LIKE 以 % 开头
WHERE name LIKE '%Tom'             -- 失效
WHERE name LIKE 'Tom%'             -- 命中

-- 4. OR 连接非索引列
WHERE name = 'Tom' OR age = 25     -- 如果 age 没索引，name 索引也失效
-- 解决：给 age 也加索引，或改用 UNION

-- 5. NOT IN / NOT EXISTS / != / <>
-- 可能导致全表扫描，取决于优化器判断
```

---

## 11.4 EXPLAIN 执行计划分析

```sql
EXPLAIN SELECT * FROM user WHERE name = 'Tom';
```

| 字段 | 说明 | 关注点 |
|------|------|--------|
| `type` | 访问类型 | system > const > eq_ref > ref > range > index > ALL |
| `key` | 实际使用的索引 | NULL 表示没用索引 |
| `rows` | 预估扫描行数 | 越小越好 |
| `Extra` | 额外信息 | Using index(覆盖索引)、Using filesort(文件排序)、Using temporary(临时表) |

**type 详解：**
- `const`：主键或唯一索引等值查询，最多一行
- `eq_ref`：关联查询中使用主键或唯一索引
- `ref`：非唯一索引等值查询
- `range`：索引范围查询（BETWEEN、>、<、IN）
- `index`：全索引扫描
- `ALL`：全表扫描（最差）

> **面试考点：怎么优化慢 SQL？**
> 1）EXPLAIN 分析执行计划，确认索引使用情况；2）检查是否有全表扫描（type=ALL）；3）优化索引：覆盖索引、联合索引；4）避免 SELECT *；5）大分页优化（延迟关联）；6）考虑分库分表。

---

## 11.5 事务与隔离级别

### 11.5.1 ACID

| 特性 | 说明 | 实现方式 |
|------|------|---------|
| 原子性（Atomicity） | 事务要么全成功，要么全失败 | Undo Log |
| 一致性（Consistency） | 事务前后数据满足约束 | 由 AID 共同保证 |
| 隔离性（Isolation） | 并发事务互不干扰 | 锁 + MVCC |
| 持久性（Durability） | 提交后数据永久保存 | Redo Log |

### 11.5.2 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|---------|------|-----------|------|---------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ | 无锁 |
| READ COMMITTED (RC) | ❌ | ✅ | ✅ | MVCC（每次读创建新快照） |
| REPEATABLE READ (RR) | ❌ | ❌ | ✅* | MVCC（事务开始时创建快照） |
| SERIALIZABLE | ❌ | ❌ | ❌ | 加锁串行执行 |

**MySQL 默认隔离级别：REPEATABLE READ**

**脏读**：读到其他事务未提交的数据
**不可重复读**：同一事务两次读同一行数据结果不同（被其他事务 UPDATE 了）
**幻读**：同一事务两次查询结果集行数不同（被其他事务 INSERT 了）

> MySQL RR 级别通过 MVCC + Next-Key Lock 基本解决了幻读问题（快照读通过 MVCC，当前读通过 Next-Key Lock）。

### 11.5.3 MVCC（多版本并发控制）

```
每行数据有两个隐藏字段：
- trx_id：最近修改该行的事务 ID
- roll_pointer：指向 Undo Log 中该行的上一个版本

Read View（读视图）包含：
- m_ids：创建 Read View 时活跃的事务 ID 列表
- min_trx_id：活跃事务中最小的 ID
- max_trx_id：下一个将分配的事务 ID
- creator_trx_id：创建该 Read View 的事务 ID

可见性判断规则：
1. trx_id == creator_trx_id → 可见（自己修改的）
2. trx_id < min_trx_id → 可见（事务已提交）
3. trx_id >= max_trx_id → 不可见（事务在 Read View 之后开启）
4. min_trx_id <= trx_id < max_trx_id：
   - 在 m_ids 中 → 不可见（事务未提交）
   - 不在 m_ids 中 → 可见（事务已提交）

RC 级别：每次 SELECT 都创建新的 Read View
RR 级别：只在事务第一次 SELECT 时创建 Read View（后续复用）
```

---

## 11.6 InnoDB 存储引擎架构

### 11.6.1 Buffer Pool（缓冲池）

```
Buffer Pool（默认 128MB，建议设为物理内存的 60%~80%）
├── 数据页（Data Page）：缓存表数据
├── 索引页（Index Page）：缓存索引数据
├── 自适应哈希索引（AHI）：自动为热点页建立哈希索引
└── Change Buffer：缓存非唯一二级索引的写操作
```

**刷脏页时机：**
1. Redo Log 写满了
2. Buffer Pool 空间不足（LRU 淘汰）
3. MySQL 空闲时后台线程定期刷
4. MySQL 正常关闭时

### 11.6.2 Redo Log（重做日志）

```
保证事务的持久性。Write-Ahead Logging（WAL）：
1. 事务修改数据时，先写 Redo Log Buffer
2. 事务提交时，将 Redo Log 刷入磁盘（顺序写，快）
3. 数据页异步刷入磁盘（随机写，慢）

即使数据页还没刷盘，MySQL 崩溃后重启也能通过 Redo Log 恢复数据。
```

**Redo Log 的组成：**
- `ib_logfile0`、`ib_logfile1`：固定大小，循环写入
- 默认每组 2 个文件，每个 48MB（MySQL 8.0.30+ 可动态调整）

### 11.6.3 Undo Log（回滚日志）

```
保证事务的原子性：
1. 事务修改前，先记录旧值到 Undo Log
2. 回滚时，根据 Undo Log 恢复旧值
3. 用于 MVCC 的版本链（roll_pointer 指向 Undo Log）
```

---

## 11.7 锁机制

### 11.7.1 锁的分类

| 分类 | 类型 | 说明 |
|------|------|------|
| 粒度 | 表锁、行锁、间隙锁、临键锁 | 粒度越小，并发越高 |
| 模式 | 共享锁（S）、排他锁（X） | S 锁之间兼容，X 锁与所有互斥 |
| 意向 | IS（意向共享）、IX（意向排他） | 快速判断表中是否有行锁 |

### 11.7.2 行锁类型

```sql
-- Record Lock（记录锁）：锁定单行记录
-- 条件列必须命中唯一索引（主键或唯一索引的等值查询）
SELECT * FROM user WHERE id = 1 FOR UPDATE;

-- Gap Lock（间隙锁）：锁定记录之间的间隙（不含记录本身）
-- 防止其他事务在间隙中插入新记录（防幻读）
-- RR 隔离级别特有
-- 例：表中有 id = [1, 5, 10]，则间隙为 (-∞,1), (1,5), (5,10), (10,+∞)

-- Next-Key Lock（临键锁）：Record Lock + Gap Lock
-- 锁定记录本身 + 记录前面的间隙
-- 左开右闭区间：(1, 5], (5, 10]
-- InnoDB 默认使用 Next-Key Lock，退化条件：
--   等值查询 + 唯一索引 → Record Lock
--   等值查询 + 非唯一索引 → Gap Lock（命中记录）或 Next-Key Lock（未命中）
--   范围查询 + 唯一索引 → Next-Key Lock
```

### 11.7.3 死锁

```sql
-- 事务 A
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 锁住 id=1
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 等待 id=2

-- 事务 B
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 2;  -- 锁住 id=2
UPDATE account SET balance = balance + 100 WHERE id = 1;  -- 等待 id=1 → 死锁
```

**InnoDB 死锁检测：**
- 主动检测：等待图（wait-for graph），发现环即死锁
- 被动检测：`innodb_lock_wait_timeout`（默认 50 秒）超时
- 检测到死锁后，回滚持有最少行锁的事务

---

## 11.8 分库分表策略

### 11.8.1 垂直拆分

```
垂直分库：按业务拆分（用户库、订单库、商品库）
垂直分表：将大表拆分为多张小表（主表 + 扩展表）
```

### 11.8.2 水平拆分

```
按某个字段将数据分散到多个表/库中：
- user_0, user_1, user_2, ... user_15
- 分片键：user_id % 16

常见分片算法：
- 取模：均匀分布，扩容麻烦
- 范围：扩容方便，可能热点
- 一致性哈希：扩容时只迁移部分数据
```

### 11.8.3 分库分表带来的问题

- **分布式事务**：用 Seata 等方案
- **跨库 JOIN**：冗余字段、应用层聚合、全局表
- **全局唯一 ID**：雪花算法、Leaf、UUID
- **分页排序**：各分片分别查询后合并排序
- **扩容迁移**：双写 → 数据校验 → 切换

---

## 11.9 读写分离与主从复制

```
主库（Master）─→ 从库1（Slave）
             ├→ 从库2（Slave）
             └→ 从库3（Slave）

主库负责写操作，从库负责读操作
```

**主从复制原理：**
1. Master 将变更写入 Binlog
2. Slave 的 IO 线程拉取 Binlog 写入 Relay Log
3. Slave 的 SQL 线程重放 Relay Log

**延迟问题：**
- 主从同步有延迟（毫秒级）
- 写后立即读可能读到旧数据
- 解决方案：关键读走主库、等待同步、半同步复制

---

## 11.10 慢 SQL 分析与优化

```bash
# 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  # 超过 1 秒记录

# 查看慢 SQL
SHOW VARIABLES LIKE 'slow_query_log_file';
```

**优化思路：**
1. EXPLAIN 分析执行计划
2. 避免 SELECT *，只查需要的字段
3. 优化索引：覆盖索引、联合索引
4. 大分页优化：`LIMIT 1000000, 10` → 延迟关联
   ```sql
   -- 慢：LIMIT 大偏移量
   SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
   -- 快：延迟关联
   SELECT u.* FROM user u
   INNER JOIN (SELECT id FROM user ORDER BY id LIMIT 1000000, 10) tmp
   ON u.id = tmp.id;
   ```
5. 避免在 WHERE 中对索引列做函数运算
6. 小表驱动大表（IN/EXISTS 选择）
7. 批量操作替代逐条操作

---

## 面试精选

### 1. InnoDB 为什么用 B+ 树不用 B 树或红黑树？

红黑树是二叉树，树高大，IO 次数多。B 树的非叶子节点也存数据，每页能存的 key 更少，树更高。B+ 树非叶子节点只存 key，每页存更多 key，树更矮（通常 3~4 层就能存千万级数据），IO 更少。叶子节点用双向链表连接，天然支持范围查询和排序。

### 2. 什么是回表？怎么避免？

通过二级索引查询时，先在二级索引中找到主键值，再通过主键在聚簇索引中找到完整行数据，这就是回表。避免方式：使用覆盖索引——查询字段全部在索引中，无需回表。例如联合索引 (name, age)，查询 `SELECT name, age FROM user WHERE name='Tom'` 无需回表。

### 3. 聚簇索引和非聚簇索引的区别？

聚簇索引的叶子节点存储完整行数据，一个表只有一个（就是主键），数据按主键顺序物理存储。非聚簇索引（二级索引）的叶子节点存储主键值，查询需要回表。InnoDB 必须有聚簇索引，如果没有主键会自动选择一个唯一索引，都没有则生成隐藏的 rowid。

### 4. MVCC 的原理？RC 和 RR 的区别？

每行数据有隐藏字段 trx_id（修改事务 ID）和 roll_pointer（指向 Undo Log 版本链）。读取时通过 Read View 判断哪个版本可见。RC 每次 SELECT 创建新 Read View（所以能看到其他事务已提交的修改）；RR 只在第一次 SELECT 时创建 Read View（所以整个事务看到的数据一致）。

### 5. 什么是 Next-Key Lock？怎么解决幻读？

Next-Key Lock = Record Lock + Gap Lock，锁定记录本身和前面的间隙，左开右闭区间。RR 隔离级别下，快照读通过 MVCC 解决幻读，当前读（SELECT FOR UPDATE）通过 Next-Key Lock 解决幻读。等值查询唯一索引时退化为 Record Lock，不会锁间隙。

### 6. MySQL 有哪些锁？

表级锁：表锁、元数据锁（MDL）、意向锁（IS/IX）。行级锁：Record Lock（记录锁）、Gap Lock（间隙锁，RR 级别防幻读）、Next-Key Lock（临键锁，默认使用）。还有插入意向锁、自增锁等。InnoDB 行锁是通过给索引上的索引项加锁实现的，没有索引时退化为表锁。

### 7. Redo Log 和 Undo Log 的区别？

Redo Log 保证持久性，记录的是"物理修改"（某页某偏移写入了什么数据），采用 WAL 机制先写日志再写数据。Undo Log 保证原子性，记录的是"逻辑修改"的逆操作（修改前的旧值），用于回滚和 MVCC 版本链。Redo Log 是顺序写（快），数据页是随机写（慢）。

### 8. 怎么优化大分页查询？

LIMIT 1000000, 10 会扫描 1000010 行再丢弃前 1000000 行。优化方案：1）延迟关联：先查主键再回表；2）游标分页：`WHERE id > last_id ORDER BY id LIMIT 10`；3）业务层限制翻页深度。

### 9. 什么情况下索引会失效？

对索引列做函数/运算、隐式类型转换、LIKE 以 % 开头、OR 连接非索引列、NOT IN/NOT EXISTS、联合索引未遵循最左前缀原则。判断方式：EXPLAIN 查看 key 是否为 NULL、type 是否为 ALL。
