---
title: Go 数据库与持久层实战
date: 2026-06-18 09:00:00
tags:
  - Java对比
  - Go
  - MySQL
  - Redis
  - MongoDB
  - GORM
  - 面试
categories:
  - Go 进阶之路
---

# 第四阶段：数据库与持久层

> 掌握 Go 生态中主流的数据库驱动与 ORM 框架，构建高性能数据访问层

---

## 18. MySQL 与 Go

### 18.1 database/sql 标准接口

```go
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql" // 匿名导入，注册驱动
)

// 连接数据库
db, err := sql.Open("mysql", "user:password@tcp(localhost:3306)/dbname?charset=utf8mb4&parseTime=true&loc=Local")
if err != nil {
    log.Fatal(err)
}
defer db.Close()

// sql.Open 不会真正建立连接，只是初始化连接池
// 验证连接
if err := db.Ping(); err != nil {
    log.Fatal(err)
}

// 连接池配置
db.SetMaxOpenConns(100)               // 最大打开连接数
db.SetMaxIdleConns(25)                // 最大空闲连接数
db.SetConnMaxLifetime(5 * time.Minute) // 连接最大存活时间
db.SetConnMaxIdleTime(3 * time.Minute) // 空闲连接最大存活时间（Go 1.15+）

// 连接池状态监控
stats := db.Stats()
fmt.Printf("Open: %d, InUse: %d, Idle: %d\n",
    stats.OpenConnections, stats.InUse, stats.Idle)
```

### 18.2 CRUD 操作

```go
// 查询单行
var user User
err := db.QueryRowContext(ctx,
    "SELECT id, name, email FROM users WHERE id = ?", id).
    Scan(&user.ID, &user.Name, &user.Email)

switch {
case errors.Is(err, sql.ErrNoRows):
    // 未找到
case err != nil:
    // 其他错误
default:
    // 成功
}

// 查询多行
rows, err := db.QueryContext(ctx,
    "SELECT id, name, email FROM users WHERE age > ?", 18)
if err != nil {
    return err
}
defer rows.Close()

var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
        return err
    }
    users = append(users, u)
}
if err := rows.Err(); err != nil { // 检查遍历过程中的错误
    return err
}

// 插入
result, err := db.ExecContext(ctx,
    "INSERT INTO users (name, email) VALUES (?, ?)", "Alice", "alice@example.com")
if err != nil {
    return err
}
id, _ := result.LastInsertId()
affected, _ := result.RowsAffected()

// 更新
_, err = db.ExecContext(ctx,
    "UPDATE users SET name = ? WHERE id = ?", "Bob", 1)

// 删除
_, err = db.ExecContext(ctx,
    "DELETE FROM users WHERE id = ?", 1)
```

### 18.3 预编译与事务

```go
// 预编译语句（防止 SQL 注入 + 性能优化）
stmt, err := db.PrepareContext(ctx,
    "INSERT INTO users (name, email) VALUES (?, ?)")
if err != nil {
    return err
}
defer stmt.Close()

_, err = stmt.ExecContext(ctx, "Alice", "alice@example.com")
_, err = stmt.ExecContext(ctx, "Bob", "bob@example.com")

// 事务
tx, err := db.BeginTx(ctx, &sql.TxOptions{
    Isolation: sql.LevelSerializable,
})
if err != nil {
    return err
}
defer tx.Rollback() // 如果 Commit 成功，Rollback 是 no-op

_, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance - ? WHERE id = ?", 100, fromID)
if err != nil {
    return err
}
_, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance + ? WHERE id = ?", 100, toID)
if err != nil {
    return err
}

return tx.Commit()
```

### 18.4 GORM

```go
import "gorm.io/gorm"
import "gorm.io/driver/mysql"

// 连接
dsn := "user:pass@tcp(localhost:3306)/dbname?charset=utf8mb4&parseTime=true"
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})

// 模型定义
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Name      string         `gorm:"size:100;not null"`
    Email     string         `gorm:"uniqueIndex;size:200"`
    Age       int            `gorm:"default:0"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"` // 软删除
}

// 自动迁移
db.AutoMigrate(&User{})

// CRUD
// 创建
db.Create(&User{Name: "Alice", Email: "alice@example.com", Age: 25})

// 查询
var user User
db.First(&user, 1)                          // 主键查询
db.Where("email = ?", "alice@example.com").First(&user)
db.Where("age > ?", 18).Find(&users)

// 更新
db.Model(&user).Update("name", "Bob")
db.Model(&user).Updates(User{Name: "Bob", Age: 30}) // 非零值字段
db.Model(&user).Updates(map[string]interface{}{"age": 0}) // map 可更新零值

// 删除
db.Delete(&user, 1) // 软删除（设置 deleted_at）

// 链式操作
db.Where("age > ?", 18).
    Order("created_at DESC").
    Limit(10).
    Offset(20).
    Find(&users)

// 关联查询
type Post struct {
    ID       uint
    Title    string
    UserID   uint
    User     User `gorm:"foreignKey:UserID"`
    Comments []Comment
}

// 预加载
db.Preload("Comments").Preload("User").Find(&posts)
db.Preload("Comments", "status = ?", "approved").Find(&posts) // 条件预加载

// Joins 加载（LEFT JOIN，一次查询）
db.Joins("User").Find(&posts)

// 钩子函数
func (u *User) BeforeCreate(tx *gorm.DB) error {
    u.CreatedAt = time.Now()
    return nil
}

func (u *User) AfterDelete(tx *gorm.DB) error {
    // 清理关联数据
    return tx.Where("user_id = ?", u.ID).Delete(&Post{}).Error
}

// 事务
db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    if err := tx.Create(&post).Error; err != nil {
        return err
    }
    return nil // commit
})

// Scope 复用查询条件
func ActiveUsers(db *gorm.DB) *gorm.DB {
    return db.Where("active = ?", true)
}
db.Scopes(ActiveUsers).Find(&users)
```

---

## 19. Redis 与 Go

### 19.1 go-redis 基础

```go
import "github.com/redis/go-redis/v9"

rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: "",
    DB:       0,
    PoolSize: 100,
})

ctx := context.Background()

// 基本操作
err := rdb.Set(ctx, "key", "value", 10*time.Minute).Err()
val, err := rdb.Get(ctx, "key").Result()
if errors.Is(err, redis.Nil) {
    fmt.Println("key 不存在")
}

// 批量操作（Pipeline，减少网络往返）
pipe := rdb.Pipeline()
pipe.Set(ctx, "key1", "v1", 0)
pipe.Set(ctx, "key2", "v2", 0)
pipe.Get(ctx, "key1")
cmds, err := pipe.Exec(ctx)

// Lua 脚本（原子操作）
script := redis.NewScript(`
    local current = redis.call('GET', KEYS[1]) or 0
    if tonumber(current) < tonumber(ARGV[1]) then
        redis.call('SET', KEYS[1], ARGV[1])
        return 1
    end
    return 0
`)
result, err := script.Run(ctx, rdb, []string{"counter"}, 100).Int()

// 发布订阅
sub := rdb.Subscribe(ctx, "channel1")
ch := sub.Channel()
for msg := range ch {
    fmt.Println(msg.Channel, msg.Payload)
}

// Stream（Go-Redis 支持 Redis Streams）
rdb.XAdd(ctx, &redis.XAddArgs{
    Stream: "mystream",
    Values: map[string]interface{}{"name": "Alice", "action": "login"},
})

// 消费消息
entries, _ := rdb.XRead(ctx, &redis.XReadArgs{
    Streams: []string{"mystream", "0"},
    Count:   10,
    Block:   5 * time.Second,
}).Result()
```

### 19.2 分布式锁实现

```go
import redsync "github.com/go-redsync/redsync/v4"
import goredis "github.com/go-redsync/redsync/v4/redis/goredis/v9"

pool := goredis.NewPool(rdb)
rs := redsync.New(pool)

// 获取锁
mutex := rs.NewMutex("my-lock",
    redsync.WithExpiry(10*time.Second),      // 锁过期时间
    redsync.WithRetryDelay(500*time.Millisecond),
    redsync.WithTries(3),
)

if err := mutex.Lock(); err != nil {
    log.Fatal("获取锁失败:", err)
}
defer mutex.Unlock()

// 执行受保护的业务逻辑
doSomething()

// 锁续期（自动续期在 WithExpiry 中配置）
```

---

## 20. 其他存储

### 20.1 MongoDB

```go
import "go.mongodb.org/mongo-driver/mongo"
import "go.mongodb.org/mongo-driver/bson"

ctx := context.Background()
client, _ := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))
defer client.Disconnect(ctx)

db := client.Database("mydb")
coll := db.Collection("users")

// 插入
coll.InsertOne(ctx, bson.M{"name": "Alice", "age": 25})

// 查询
var result bson.M
coll.FindOne(ctx, bson.M{"name": "Alice"}).Decode(&result)

// 查询多个
cursor, _ := coll.Find(ctx, bson.M{"age": bson.M{"$gt": 18}})
defer cursor.Close(ctx)
for cursor.Next(ctx) {
    var user bson.M
    cursor.Decode(&user)
}

// 更新
coll.UpdateOne(ctx,
    bson.M{"name": "Alice"},
    bson.M{"$set": bson.M{"age": 26}},
)

// 使用结构体标签
type User struct {
    ID    primitive.ObjectID `bson:"_id,omitempty"`
    Name  string             `bson:"name"`
    Age   int                `bson:"age"`
    Email string             `bson:"email"`
}
```

### 20.2 嵌入式存储

```go
// BoltDB —— B+ 树结构的 KV 存储（只读并发安全）
import bolt "go.etcd.io/bbolt"

db, _ := bolt.Open("my.db", 0600, nil)
defer db.Close()

db.Update(func(tx *bolt.Tx) error {
    b, _ := tx.CreateBucketIfNotExists([]byte("users"))
    b.Put([]byte("user:1"), []byte(`{"name":"Alice"}`))
    return nil
})

db.View(func(tx *bolt.Tx) error {
    b := tx.Bucket([]byte("users"))
    v := b.Get([]byte("user:1"))
    fmt.Println(string(v))
    return nil
})

// BadgerDB —— LSM 树结构，写入性能更好
import "github.com/dgraph-io/badger/v4"

db, _ := badger.Open(badger.DefaultOptions("/tmp/badger"))
defer db.Close()

db.Update(func(txn *badger.Txn) error {
    return txn.Set([]byte("key"), []byte("value"))
})
```

---

## 面试专题

**Q1：database/sql 的连接池是如何工作的？**
- sql.Open 创建连接池，不会立即建立连接
- 每次 Query/Exec 时从池中取连接，用完归还
- MaxOpenConns 控制总连接数，MaxIdleConns 控制空闲数
- ConnMaxLifetime 控制连接最大存活时间，防止使用过期连接

**Q2：GORM 的软删除是如何实现的？**
- 在模型中嵌入 `gorm.DeletedAt` 字段
- Delete 操作变为 `UPDATE SET deleted_at = now()` 而非真删除
- 查询时自动添加 `WHERE deleted_at IS NULL` 条件
- 使用 `Unscoped()` 可以查询已软删除的记录

**Q3：Redis Pipeline 为什么快？**
- 批量发送多个命令，减少网络往返（RTT）次数
- N 个命令从 N 次往返变成 1 次往返
- 服务端批量执行并依次返回结果

**Q4：如何选择 GORM 和 database/sql？**
- GORM：开发效率高，自动迁移、钩子、关联查询、scope 等
- database/sql + sqlx：性能更好，SQL 完全可控，适合复杂查询
- 建议：简单 CRUD 用 GORM，复杂 SQL 用 sqlx，两者可混用

---

## 推荐资源

- [go-sql-driver/mysql 文档](https://github.com/go-sql-driver/mysql)
- [GORM 官方文档](https://gorm.io/docs/) —— 中文文档完善
- [go-redis 文档](https://redis.uptrace.dev/)
- [sqlx 文档](https://github.com/jmoiron/sqlx)
