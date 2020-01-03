---
layout: post
title: "值传递和引用传递"
tag: "transmit"
date: "2018-04-03 00:00:00 +0800"
categories: "java"
---

### 到底什么是值传递和引用传递  


基础中的基础，首先不要纠结字面意思，Java中（byte、short、int、long、float、double）、字符型（char）、布尔型（boolean）这些基本类型数据直接保存在栈中，而类、
接口、数组数据是保存在堆中，栈只是保存一个指向堆内存的指针。  

<!--more-->

值传递和引用传递的本质就是**传递值还是内存地址**
