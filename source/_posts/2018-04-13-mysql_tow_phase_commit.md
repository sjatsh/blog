---
layout: post
title: "mysql bin/redo log顺序一致性问题"
tag: "mysql"
date: "2018-04-13 00:00:00 +0800"
categories: "db"
---

### 为什么mysql既有bin log也有redo log

bin log是属于mysql server层的log，主要是用来做**主从复制和即时点恢复**时使用的，**redo log是InnoDB存储引擎层的，用来保证事务安全**。
不管用哪种存储引擎都有bin log，而redo log只有InnoDB有，这是由mysql体系架构决定的。

<!--more-->

### bin log

mysql5.1之后有bin log，主要用于主备复制。

#### 未开启bin log

未开启bin log情况下，InnoDB通过redo log、undo log进行数据的回复和事务回滚，以此保证CrashSafe。可以通过`innodb_flush_log_at_trx_commit`
来控制何时写入redo log，用户可以根据需求来自行调整。`innodb_flush_log_at_trx_commit = 0|1|2`

- 0：每N秒将Redo Log Buffer的记录写入Redo Log文件，并且将文件刷入硬件存储1次。N由innodb_flush_log_at_timeout控制。
- 1：每个事务提交时，将记录从Redo Log Buffer写入Redo Log文件，并且将文件刷入硬件存储。
- 2：每个事务提交时，仅将记录从Redo Log Buffer写入Redo Log文件。Redo Log何时刷入硬件存储由操作系统和innodb_flush_log_at_timeout
决定。这个选项可以保证在MySQL宕机，而操作系统正常工作时，数据的完整性。

crash recovery时，已经在InnoDB内部提交的事务用redo log恢复，所有prepare但是没有commit的transactions用undo log做rollback。

#### 开启bin log

在开启bin log的情况下，要保证主备数据一致性就必须保证bin log和redo log日志的一致性，mysql引入两阶段提交（two phase commit or 2pc）。
mysql内部会将一个普通的事物作为一个XA事物（分布式事物）来处理。

- 为每一个事务分配一个XID
- commit分为prepare和commit两个阶段
- bin log作为事务的协调者(Transaction Coordinator)，bin log event作为协调者日志

![](https://olef5l6y5.qnssl.com/20180415235500.png)

- prepare阶段：sql成功执行，生成`xid、redo log、undo log`。调用prepare将事务状态设为TRX_PREPARED，并将redo log、undo log刷磁盘
- commit阶段：  
  - 记录bin log，write()&fsync()，只要fsync()成功事务就一定要提交了，此时生成`Xid_log_event`，失败调用`ha_rollback_trans`回滚。
  - InnoDB commit完成事务提交，清除undo log、刷redo log日志，设置事务状态为`TRX_NOT_STARTED`
  
两阶段提交通过`innodb_support_xa`控制，默认true开启。为了保证CrashSafe，上面fsync可以通过`sync_binlog=1&innodb_flush_log_at_trx_commit=1`
保证。

innodb_flush_log_at_trx_commit(redo log)

- 0：每秒一次写入log file，并且fsync。
- 1：每次事务提交写入log file并fsync（默认）。
- 2：每次事务提交写入log file不fsync，何时fsync由操作系统决定。

sync_binlog(binlog)

- 0：何时fsync由OS决定。
- N：N次事务提交后fsync。

### crash recovery

开启bin log的mysql在crash recovery时找出所有prepare阶段生成的`xid`并找出对应的`Xid_log_event`

- Xid_log_event存在，事务提交
- Xid_log_event不存在，事务回滚

从上面可以看出，只要是bin log fsync()刷盘成功生成`Xid_log_event`此时事务必定commit成功。

### bin log、redo log顺序一致

两阶段提交只能保证bin log、redo log数据的一致性，但是却不能保证bin log与redo log的存储顺序以直性。但是在并发场景下却无法保证所有事物在
bin log、redo log中提交顺序的一致性。当我们使用xtrabackup或ibbackup进行数据备份时就可能造成数据的不一致，如图：
![](https://olef5l6y5.qnssl.com/20180416002900.png)

5.6之前通过`prepare_commit_mutex`来保证XA事务的顺序一致性，每次只有一个XA事务能够获得该锁。

![](https://olef5l6y5.qnssl.com/20180416003200.png)

上图所示MySQL开启Binary log时使用`prepare_commit_mutex`保证二进制日志和存储引擎顺序保持一致，`prepare_commit_mutex`的锁机
制造成高并发提交事务的时候性能非常差而且bin log也无法`group commit`。

mysql5.6之后引入三阶段提交，移除`prepare_commit_mutex`，并且`bin log&redo log`都是`group commit`。[MySQL组提交](http://www.ywnds.com/?p=5798)


