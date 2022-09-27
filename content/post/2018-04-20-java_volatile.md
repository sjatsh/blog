---
layout: post
title: "Java 内存可见性与指令重排"
tag: "volatile"
date: "2018-04-20 00:00:00 +0800"
categories: "技术"
---

###  内存可见性的问题

多线程系统中共享变量在主存，线程保存一份变量的副本，某个线程对共享变量的改动通过改动变量副本然后同步到主存中的共享变量，但是其他线程保存的可能还是之前的变量副本，此时可能
就会存在问题。  

<!--more--> 

全局变量open：
```
boolean open=true;
```
线程A经过一些操作之后把open设置成false：
```
//线程A
open = false;
resource.close();
```
此时线程B通过判断open状态进行资源访问：
```
//线程B
while(open) {
doSomethingWithResource(resource);
}
```
此时将open置为false后关闭资源，但是此时open变量可能并没有同步到线程B，此时open对线程B不可见。但是实际资源已经被关闭，此时再对已关闭的资源进行操作就会产生问题。

#### volatile提供内存可见性

volatile可见性原理是在**每次访问变量时都会进行一次刷新**，因此每次访问都是主内存中最新的版本。所以volatile关键字的作用之一就是保证变量修改的实时可见性。

### 指令重排问题

指令重排导致单例模式失效：

```
public class Singleton {
  private static Singleton instance = null;
  private Singleton() { }
  public static Singleton getInstance() {
     if(instance == null) {
        synchronzied(Singleton.class) {
           if(instance == null) {
               instance = new Singleton();  //非原子操作
           }
        }
     }
     return instance;
   }
}
```
其中`instance= new Singleton()`并不是原子操作，实际上会被抽象成几个JVM指令：
```
memory = allocate();    //1：分配内存空间
ctorInstance(memory);   //2：初始化对象
instance = memory;      //3：instance引用指向内存空间
```
其中1、2，1、3相互依赖，但是2、3并不是相互依赖，JVM可以对他们进行指令优化重排：
```
memory = allocate();    //1：分配内存空间
instance = memory;      //2：instance引用指向内存空间
ctorInstance(memory);   //3：初始化对象
```
由于instance引用在第二步已经指向内存所以不为空，但是还没有对对象进行初始化操作，这时`getInstance`判断instance不为空拿去使用就会导致出错。


#### 内存屏障

内存屏障（Memory Barrier，或有时叫做内存栅栏，Memory Fence）是一种CPU指令，用于控制特定条件下的重排序和内存可见性问题。Java编译器也会根据内存屏障的规则禁止重排序。
内存屏障可以被分为以下几种类型  

- LoadLoad屏障：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
- StoreStore屏障：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。
- LoadStore屏障：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
- StoreLoad屏障：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。
它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。
![](https://olef5l6y5.qnssl.com/20180423135300.png)

为了保证final字段的特殊语义，也会在下面的语句加入内存屏障：  
x.finalField = v; StoreStore; sharedRef = x;

#### final域重排序

在写final域的时候有两个规则：

- JVM禁止编译器把final域的写重排序到构造函数之外
- 编译器会在final域的写之后，构造函数return之前，插入一个**StoreStore**屏障，这个屏障禁止处理器把final域的写重排序到构造函数之外。

写final域的重排序规则可以确保:**在对象引用为任意线程可见之前,对象的final域已经被正确初始化过了**。




