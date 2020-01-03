---
layout: post
title: "mysql index"
description: "mysql索引"
tag: "mysql"
date: "2018-03-24 00:00:00 +0800"
categories: "db"
---

### 什么是数据库索引（Mysql）

用了这么久Mysql，Mysql索引相关知识点却不太清楚，可以说匮乏。正好看到有面试问道，稍微看看和整理一下。  

<!--more-->

- 索引（index）是存储引擎（storage engine）的实现，不是server。不是所有存储引擎都支持所有的索引类型。
即使多个存储引擎支持某一索引，它们的实现和行为也可能有所不同。  

- Mysql官方定义：帮助Mysql高效获取数据的数据结构,简单来说索引是**一种数据结构**，用以加快数据的查询速度。

### 索引的分类

#### 数据结构角度

- B+树索引(O(log(n)))：关于B+树索引

- hash索引  
    - 只能满足相等查询，不能使用范围查询  
    - 效率高、一次定位，不像B-Tree 索引需要从根节点遍历，Hash 索引的查询效率要远高于B-Tree  
    - 只有Memory存储引擎显示支持hash索引  

- FULLTEXT索引（MyISAM和InnoDB引擎都支持了）

- R-Tree索引（用于对GIS数据类型创建SPATIAL索引）  

#### 存储角度  

- 聚集索引（clustered index）

- 辅助索引（non-clustered index）

[参考](https://www.jianshu.com/p/2879225ba243)

#### 逻辑角度  

- 主键索引：主键索引是一种特殊的唯一索引，不允许有空值

- 普通索引或者单列索引

- 多列索引（复合索引）：复合索引指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。[最左前缀原则](http://www.ywnds.com/?p=8754)

- 唯一索引

- 空间索引（MYISAM）：空间索引是对空间数据类型的字段建立的索引，MYSQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING、POLYGON。
MYSQL使用SPATIAL关键字进行扩展，使得能够用于创建正规索引类型的语法创建空间索引。创建空间索引，必须声明为NOT NULL，只能在MYISAM的表中创建  

### 主键索引和普通索引的区别  

特殊的唯一索引，不允许有空值，唯一索引可以为空。普通索引或单列索引只是字段数据单独使用数据结构存储加快查询速度，没有其他特殊约束。

### 有索引但不生效的情况  

- 如果条件中有or，即使其中有部分条件带索引也不会使用，要想使用or，又想让索引生效，只能将or条件中的每个列都加上索引。
- 对于多列索引，满足断点前生效规则，[参考](https://www.cnblogs.com/codeAB/p/6387148.html)。
- like查询是以%开头。
- 存在数据类型隐形转换（如：字段是字符串类型，查询时不加引号）。
- where条件对字段有数学运算。
- where条件对字段使用函数。
- mysql判断使用全表扫描要比使用索引时（如：数据量极少时）。

### 不推荐使用索引的情况  

- 数据唯一性差（取值只有几种，意味着索引树级别少，多是平级，无异于全表扫描）
- 频繁更新的字段不要使用索引（频繁变化导致索引也频繁变化，增大数据库工作量，降低效率）
- 字段不在where语句出现时
- where后含IS NULL / IS NOT NULL/ like '%**%' 等条件时
- where 子句里对索引列使用不等于（<>），使用索引效果一般

### 覆盖索引（covering index）  
covering index 提升查询效率：MySQL只需要通过索引就可以返回查询所需要的数据，不必回表查询。
[参考](http://webcache.googleusercontent.com/search?q=cache:http://flacro.me/covering-index/)

### 优化经验  

参考：[https://blog.csdn.net/kaka1121/article/details/53395587](https://blog.csdn.net/kaka1121/article/details/53395587)  
参考：[http://www.cnblogs.com/hongfei/archive/2012/10/19/2731342.html](http://www.cnblogs.com/hongfei/archive/2012/10/19/2731342.html)