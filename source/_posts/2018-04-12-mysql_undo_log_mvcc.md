---
layout: post
title: "mysql undo log与MVCC的实现原理"
tag: "mysql"
date: "2018-04-12 00:00:00 +0800"
categories: "db"
---

###mysql架构 

概念上可以分为四层：

- 接入层：不同客户端通过mysql协议与mysql连接通讯，接入层进行权限验证、连接池管理线程管理等。
- 服务层：sql解析器、优化器、数据缓冲、缓存等。
- 存储引擎：常用的`MyISAM、InnoDB`等[存储引擎](https://my.oschina.net/RubinWan/blog/87031)。
- 文件系统：保存数据、索引、日志等。

<!--more-->

![](https://olef5l6y5.qnssl.com/20180412101558.jpg)

### MVCC

MVCC(Multi Version Concurrency Control)多版本并发控制。  
InnoDB可重复读隔离级别中**聚集索引叶子节点中保存了回滚指针，指向修改前的存放在undo log中的数据**，InnoDB也正是通过**回滚指针和undo log实现MVCC和事务回滚**。  

**undo log中的内容只是串行化的结果，记录了多个事务的过程，不属于多版本共存**。但理想的MVCC是难以实现的，当事务仅修改一行记录使用理想的MVCC模式是没有问题的，
可以通过比较版本号进行回滚，但当事务影响到多行数据时，理想的MVCC就无能为力了。  

**问题**：transaction1修改row1、row2，row1修改失败row2修改成功，此时事务需要回滚row1、row2不需要回滚，此时row1上是没有锁的并且可能被transaction2修改，
但是我们现在回滚row1的内容就可能造成破坏transaction2的结果。  

**思考**：理想MVCC难以实现的根本原因在于企图通过乐观锁代替二段提交。**修改两行数据，但为了保证其一致性，与修改两个分布式系统中的数据并无区别**，而二提交是目前这种场景
保证一致性的唯一手段。

- 二段提交的本质是锁定
- 乐观锁的本质是消除锁定，本质上存在矛盾 
- 理想的MVCC难以实现，Innodb只是借了MVCC这个名字，提供了非阻塞读
  
### redo/undo/bin log

- redo log：内存中的修改操作都会记录在redo log中，当内存和磁盘中的数据不一致或事物提交时会进行一次flush操作。
- undo log：记录数据的反向操作，用于操作的撤回和数据的恢复，**通过undo log可以实现事务回滚，并且可以根据undo log回溯到某个特定的版本，实现MVCC**。
- bin  log：**binlog是mysql服务层产生的日志**，常用来进行**数据恢复、数据库复制**，常见的mysql主从架构，就是采用slave同步master的binlog实现的, 另外通过解析binlog
能够实现mysql到其他数据源（如ElasticSearch)的数据复制。

### redo/bin log一致性保证 

为了防止写完bin log但是redo log的事务没提交从而导致的不一致，innodb 使用了两阶段提交，其基本逻辑就是上锁：

```
InnoDB prepare write/sync redo/undo log (prepare_commit_mutex mysql5.6前)
write/sync Binlog
InnoDB commit (写入COMMIT标记后释放prepare_commit_mutex)
```







