---
layout: post
title: "why codis not redis cluster"
date: "2018-04-04 00:00:00 +0800"
categories: "redis"
---

### why not redis cluster  

- 无中心的设计，很难把程序写对  
- 代码有点吓人，clusterProcessPacket有426行，大脑很难处理到所有的状态切换
- 迟迟没有正式版，等了4年之久
- 缺乏Best Practise，还没人写一个Redis Cluster的若干条注意事项
- 这个系统高度耦合，升级困难  
- 如果slot所在的redis master、slave同时挂掉，整个集群不可用。codis是否一样?待验证...

<!--more-->

## why codis 

codis改进redis的几个点:  

- redis 数据量太大的话（22G 以上），处理性能就开始下降
- 无法区分冷热数据，内存浪费严重
- RDB Block 住整个服务
- 写操作太频繁，AOF刷盘太多，很容易rewrite 

[https://segmentfault.com/a/1190000012908434](https://segmentfault.com/a/1190000012908434)