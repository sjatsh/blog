---
layout: post
title: "mysql InnoDB locks"
tag: "mysql"
date: "2018-04-11 00:00:00 +0800"
categories: "技术"
---

###InnoDB的锁机制

数据库锁为了支持更好的并发，提供数据的一致性和完整性。锁的类型有：共享锁（S）、排他锁（X）、意向共享（IS）、意向排他（IX）。为了更好的
并发，InnoDB通过MVCC提供非锁定读：不等待行上的锁释放，读取行的一个快照。

<!--more-->

### InnoDB支持的锁定方式

- Record Lock：记录锁，锁直接加在索引记录上面，锁住的是key而不是记录值。
- Gap Lock：间隙锁，锁的一个范围，不包括记录本身，防止其他事务的插入操作，以此防止幻读的发生。
- Next-Key Lock：锁定一个范围，并且锁定记录本身。

### 锁定方式的选择

- 如果条件没有走索引，会进行全表扫描，所以上升为表锁。
- 如果条件为索引字段，但是并非唯一索引（包括主键索引），那么此时会使用`Next-Key Lock`，为什么用`Next-Key Lock`：
  - 符合条件的记录上加上排他锁，会锁定当前非唯一索引和对应的主键索引的值
  - 保证锁定的区间不能插入新的数据
  - 如果条件为唯一索引，使用`Record Lock`  