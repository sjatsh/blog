---
layout: post
title: "mysql transaction isolation"
date: "2018-04-09 00:00:00 +0800"
tag: "mysql"
categories: "技术"
---

###何为隔离级别

**隔离级别决定事务间的可见程度，一个事务的修改结果在什么时候能被其他事务看到**，由此SQL1992规范对隔离性定义了不同的**隔离级别**用来划
分这个可见范围。
<!--more-->

### mysql四种事务隔离级别

SQL1992中定义的四种事务隔离级别：`READ UNCOMMITTED、READ COMMITTED、REPEATABLE READ、SERIALIZABLE`，mysql InnoDB默认
`REPEATABLE READ`，mysql可以用`transaction-isolation`或者在配置文件中可以为所有连接设置隔离搞我winiseI

- my.ini设置：

```
transaction-isolation = {READ-UNCOMMITTED | READ-COMMITTED | REPEATABLE-READ | SERIALIZABLE}
```

- `SET TRANSACTION`设置：

```
SET [SESSION|GLOBAL] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED|READ COMMITTED|REPEATABLE READ|SERIALIZABLE}
```
`global`设置全局事务隔离级别，`session`设置session

查询全局或者当前session隔离级别：

```
SELECT @@global.tx_isolation;
SELECT @@session.tx_isolation;
SELECT @@tx_isolation;
```

| 隔离级别 | 脏读（Dirty Read） | 不可重复读（NonRepeatable Read） |  幻读  |
| :--- | :---- | :---- | :---- |
| 未提交读（Read uncommitted）| 可能 | 可能 | 可能 |
| 已提交读（Read committed） | 不可能 | 可能 | 可能 |
| 可重复读（Repeatable read） | 不可能 | 不可能 | 可能 |
| 可串行化（Serializable ） | 不可能 | 不可能 | 不可能 |

隔离级别：

- 未提交读：允许脏读，也就是可能读取到其他会话中未提交事务修改的数据
- 已提交读：只能读取到已经提交的数据。Oracle等多数数据库默认都是该级别 (不重复读)
- 可重复读：可重复读。在同一个事务内的查询都是事务开始时刻一致的，InnoDB默认级别。在SQL标准中，该隔离级别消除了不可重复读，但是还存在幻象读
- 可串行化：完全串行化的读，每次读都需要获得表级共享锁，读写相互都会阻塞

现象/结果：

- 脏读：当前事务可以读取和使用其他事务中未提交的数据。
- 不可重复读：一个事务中由于另一个事务对同一数据造成修改，从而导致事务内两次读取到的数据不一致。
- 幻读：幻读可能的情况很多
  - 第一个事务中在插入数据前先查询发现数据不存在，接下来准备执行insert，此时事务二中执行相同操作并提交。此时事务一中insert时就会报错：
  数据已存在。就好像产生了幻觉一样明明查询的时候没有该数据，但是insert的时候却报错已存在。
  - 或者第一个事务对全表执行update操作，此时事务二insert了一条数据，事务一用户之后就会发现表中还会有没有更新的数据，好像产生幻觉。

当事务隔离级别是**可重复读**的时候，并且`innodb_locks_unsafe_for_binlog=off`（默认），搜索index并加行锁的时候使用`next-key locks`
可以有效避免幻读。但这里需要注意**加行锁字段必须是索引字段，否则会表锁。**












