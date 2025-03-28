+++

date = '2025-03-04T11:51:53+08:00'
draft = true
title = 'Rookie DB'
description = "a bare-bones database implementation"
image = "derpydb-small.jpg"
categories = ["database", "sql", "B+ tree", "concurrency", "recovery"]

+++

这个项目包含一个基本的数据库实现, 项目框架放在[这里](), 我的实现在[这里]().

本项目中, 我将添加数据库对B+树索引, 高效连接算法, 查询优化, 多粒度锁定(以支持并发事务执行)和数据库恢复的支持.

## B+ Trees

索引的定义: An index is data structure that enables fast lookup and modification of  data entries by search key.

如果没有索引, 我们在查询数据时, 数据库会为我们做全表扫描, 找到对应的数据. 建立索引的过程其实就是全表扫描后把对应的数据在磁盘上的位置记录在索引中, 从而在下一次查看相应数据时可以直接读取.

在这个项目中, 我们将实现一个B+ Tree用于创建索引. 以下是这部分的结构图:

![B+Tree_structure](B+Tree_structure.jpg)

在实现完上述功能之后, 我们运行

```sql
CREATE INDEX on Students(sid);
```

就会在Students.sid上创建一个索引, 理论上可以加快查询速度.

### 实现

![proj1](proj1.jpg)

##### fromBytes

从磁盘或存储中加载一个 **B+ 树的节点**

##### get

获得对应键的叶子节点

##### getLeftmostLeaf

获得最左边的叶子节点

##### put

添加叶子节点

##### remove

移除叶子节点

##### scanAll

获得B+tree储存的所有RecordID(RecordID用于检索page上的数据)

##### scanGreaterEqual

获得B+tree储存的大于某个值的RecordID

##### bulkLoad

将RecordID批量存储到B+tree索引中

## Questions

* q: 一个数据库的查询语句执行有哪些环节, 什么时候会用到索引?
* q: 我的**B+tree**索引的性能怎么样, 在leafNode和innerNode中都调用sync()感觉性能不好
