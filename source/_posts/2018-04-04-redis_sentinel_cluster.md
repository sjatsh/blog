---
layout: post
title: "redis sentinel/cluster"
tag: "redis"
date: "2018-04-04 00:00:00 +0800"
categories: "redis"
---

### 单例模式    

redis单例存在的问题： 

- 单点故障 
- 数据备份
- 数据量大了以后的性能问题  

<!--more-->

### 主从模式  

主从模式指的是使用一个redis实例作为master，其余的实例作为slave,主机和从机的数据完全一致，主机支持数据的写入和读取等各项操作，而从机则只支持与主机数据的同步和读取
（也可以设置可写，但是会在master同步后丢失），也就是说，客户端可以将数据写入到主机，由主机自动将数据的写入操作同步到从机。  
主从模式很好的解决了数据备份问题，并且由于主从服务数据几乎是一致的，因而可以将写入数据的命令发送给主机执行，而读取数据的命令发送给不同的从机执行，从而达到读写分离的目的。  
如下所示主机redis-A分别有redis-B、redis-C、redis-D、redis-E四个从机：
  
![](https://olef5l6y5.qnssl.com/2018-4-4-00-30-51.png)
    
**注意**：  

- 主从模式主要是为了解决上面后两个问题而提供的一种解决方案，但是依然存在单点故障的问题。
- master down掉会导致slave也不可用，可手动选择一个slave升格为master，使系统能够继续提供服务。然而整个过程很麻烦需要人工介入，难以实现自动化。 

### sentinel模式  

为了解决主从模式master down掉会导致slave也不可用，需要人工接入的问题，redis2.8提供sentinel工具，**实现自动化的系统监控和故障恢复功能**
sentinel的作用就是监控Redis系统的运行状况：

- 监控主数据库和从数据库是否正常运行。
- 主数据库出现故障时自动将从数据库转换为主数据库。  

![](https://olef5l6y5.qnssl.com/20180404004409)

可以用`info replication`查看主从情况   

**注意**：  

sentinel 1 很多 Bug，被官方弃用，用 redis2.8 以及 sentinel 2  

主从切换配置要加上  

```
sentinel can-failover server1 yes
```  

### cluster模式  

及时使用sentinel每个redis实例也是全量存储，浪费内存并且会有木桶效应。  
集群至少需要3主3从  
修改每个实例的配置文件：
- cluster-enabled yes 
- cluster-config-file nodes-xx.conf  

使用redis自带的集群工具创建集群：  

- 安装rubygems：yum install ruby 、yum install rubygems 、gem install redis
- 分别启动6个redis实例
- redis-trib.rb create --replicas 1 127.0.0.1:6379 ... 

其中replicas 1表示给每个master分配多少个slave  

### 总结  

**sentinel**    

- 主从切换自动化 
- 本身也支持集群，用sentinel集群来监控redis集群 
- 自身也需要多数，避免单点问题
- 只是一个运行在特殊模式下的 Redis实例 
- 可以通过发布与订阅来自动发现正在监视相同主实例的其他Sentinel
- failover 过程对代码客户端是透明的 

**cluster**

- 一个redis集群包含16384个哈希槽（hash slot）
- redis集群对节点使用了主从复制功能，提高稳定性（HA）
- redis集群的节点间通过gossip协议通信

### 要点  

- **主从模式是为了HA**
- sentinel是一种自动**failover和高可用（HA）解决方案**
- cluster是一种**分片的方案**，自带failover，**主要解决的是在线动态横向扩容**
