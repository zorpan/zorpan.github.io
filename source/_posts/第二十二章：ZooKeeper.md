---
title: ZooKeeper 原理与分布式协调
date: 2026-06-19 12:00:00
tags:
  - ZooKeeper
  - ZAB
  - 分布式锁
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 22.1 ZAB 协议与选举机制

```
ZAB（ZooKeeper Atomic Broadcast）协议：
保证崩溃恢复时已提交事务不丢失，以及原子广播时消息的顺序一致性

三种角色：
- Leader：处理写请求，发起提案投票
- Follower：处理读请求，参与投票
- Observer：处理读请求，不参与投票（提升读性能）

选举过程（Leader Election）：
1. 每个节点投票给自己，zxid 最大的优先，zxid 相同时 myid 最大的优先
2. 收到其他节点的投票，比较 zxid 和 myid
3. 更新投票为 zxid 最大的节点
4. 超过半数同意则当选 Leader
```

---

## 22.2 数据模型与 Watcher 机制

```
ZNode 数据模型（树形结构）：
/
├── /services
│   ├── /services/user-service
│   │   ├── /services/user-service/instance-1
│   │   └── /services/user-service/instance-2
│   └── /services/order-service
└── /config
    └── /config/database

ZNode 类型：
- 持久节点：手动删除才消失
- 临时节点：会话结束自动删除（分布式锁用）
- 顺序节点：自动递增编号（选举/队列用）
```

**Watcher 机制：**
```java
// 注册 Watcher
zk.getData("/config/db", event -> {
    if (event.getType() == Event.EventType.NodeDataChanged) {
        // 数据变更，重新加载配置
        reloadConfig();
    }
}, null);

// Watcher 是一次性的，触发后需要重新注册
```

---

## 22.3 ZooKeeper 典型应用场景

| 场景 | 实现方式 |
|------|---------|
| 分布式锁 | 临时顺序节点 + Watcher |
| 配置管理 | 持久节点 + Watcher 监听变更 |
| 服务注册 | 临时节点（服务下线自动删除） |
| 选主 | 临时顺序节点（最小节点为 Leader） |
| 分布式队列 | 顺序节点 |

---

## 面试精选

### 1. ZooKeeper 的 ZAB 协议？

ZAB 保证分布式事务的原子性和顺序一致性。写请求由 Leader 处理，Leader 将提案广播给 Follower，超过半数确认后提交。崩溃恢复时通过选举产生新 Leader。

### 2. ZooKeeper 怎么实现分布式锁？

创建临时顺序节点，判断自己是否是最小节点（是则获得锁），否则 Watcher 监听前一个节点。释放锁时删除临时节点，下一个节点收到通知尝试获取锁。避免惊群效应。

### 3. ZooKeeper 和 Eureka 的区别？

ZooKeeper 保证 CP（强一致性），选举期间不可用。Eureka 保证 AP（可用性），节点之间对等复制，可能数据不一致。ZooKeeper 适合对一致性要求高的场景，Eureka 适合对可用性要求高的场景。
