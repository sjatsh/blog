---
title: "Java类加载器"
description: "Java类加载器"
tags: "classloader"
date: "2018-04-25 00:00:00 +0800"
categories: "java"
---  

### 基本概念

最初为了满足Java Applet的需要而被开发出来的，Java Applet 需要从远程下载Java类到浏览器执行，现在Web容器和OSGi中得到了广泛的使用。类加载器负责读取 Java 字节代码，并转换
成 java.lang.Class 类的实例。每个 java.lang.Class 实例用来表示一个Java类，通过 newInstance() 创建该类对象，newInstance()只能调用无参构造器。
<!--more-->
除此之外，ClassLoader 
还负责加载 Java 应用所需的资源，如图像文件和配置文件等。

除了`BootstrapClassLoader`是由C++或其他语言编写，其他都是 java.lang.ClassLoader 的实例，ClassLoader 中与加载类相关的方法：

| 方法 | 说明 |
| :--- | :---- |
| getParent() | 返回该类加载器的父类加载器。 |
| loadClass(String name) | 加载名称为name的类 | 
| findClass(String name) | 查找名称为 name的类 | 
| findLoadedClass(String name) | 查找名称为 name的已经被加载过的类 | 
| defineClass(String name, byte[] b, int off, int len) | 把字节数组 b中的内容转换成 Java 类 |
| resolveClass(Class<?> c) | 链接指定Java类 |

**注意**：Class.forName(className)加载class的同时会初始化静态代码块，ClassLoader.loadClass(className)则不会初始化静态块。

### bootstrap class loader

一切从bootstrap class loader开始，bootstrap class loader启动加载`jre/lib/`下的`rt.jar、resources.jar、charsets.jar`等核心类库。之后加载`ExtClassLoader、
AppClassLoader`

- **BootstrapClassLoader**: 由c++实现，负责在虚拟机启动时加载Jdk核心类库以及加载后两个类加载器。
- **ExtClassLoader**: Launcher子类，继承自URLClassLoader，负责加载{JAVA_HOME}/jre/lib/ext/目录下的jar包。
- **AppClassLoader**: Launcher子类，继承自URLClassLoader，负责加载应用程序classpath目录下的所有jar和class文件。

`ExtClassLoader、AppClassLoader` 最终还是调用`ClassLoader`中的native方法，`defineClass0、defineClass1、defineClass2`。

**注意**：经常看资料说的类载入器的层级体系(classloader hierarchy)，并不是我们一般认为的层级体系（父子类继承的层级体系）。类别载入器的层级体系另有所指，不要混淆。

`ExtClassLoader、AppClassLoader`其实都是由`BootstrapClassLoader`加载，他们调用getClassLoader()返回都是null表示加载他们的都是BootstrapClassLoader，
只不过AppClassLoader设置其parent是`ExtClassLoader`。

### 类载入器的双亲委派模式

上面说的类别载入器的层级体系就是为了实现双亲委派模式，所谓双亲委派模式就是**类别载入器在载入类的时候，会首先请示 Parent 使用 Parent 搜索路径帮忙载入，如果 Parent 找不到，
那么才用自己的搜索路径去搜索加载类**

**注意**：需要注意的是，如果我们自己的Main.class调用了Test.class中的方法，但是Main.class放在 Parent 加载器搜索路径中，但是Test.class放在子类加载器搜索路径或其他路径，
就会出现NoClassDefFoundError，这也是Java的安全策略。

### 类载入器的功用

从下面这张图来分析：  
![](https://olef5l6y5.qnssl.com/20180425200900.png)

类载入器是一个非常复杂的系统，为什么要设计这么一个复杂的系统呢？肯定是有道理的，除了动态性实现动态加载以外，其实更重要的莫过于安全性。

这张图说明了两件事：

- **第一**：假设我们用URLClassLoader到网上随便下载一个class，这个class不可能是`AppClassLoader、ExtClassLoader`或者是`Bootstrap Loader`路径下的同名类（指全路径），
因此蓄意破坏者没有办法植入代码到我们的系统，除非直接黑如你的电脑替换文件，这其实也不是Java考虑的范围了，这是操作系统安全的该考虑的。
- **第二**：类载入器无法看到同级别类载入器载入的类，除了意味着不同的类別载入器可以载入完全相同的类之外，也排除了**误用**或**使用別人的恶意代码**的机会。

### 疑问

上面说到一个类由一个载入器加载，他调用的方法所在的类也必须在该载入器的搜索路径下。但是就会存在一个问题，JDBC是怎么实现的？**java.sql.Driver**属于基础类库，由
BootstrapClassLoader加载，但是JDBC Driver的实现是由各个厂商实现的，按照上面的逻辑BootstrapClassLoade应该会在**jre/lib/**下面找对应实现类，但是很明显我们的驱动的
jar包是在classpath下面，JDBC的实现后续继续再讨论。







