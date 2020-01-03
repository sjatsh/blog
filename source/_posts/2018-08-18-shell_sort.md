---
layout: post
title: "突破算法之-希尔排序"
tag: "sort"
date: "2018-08-18 20:24:00 +0800"
categories: "algorithms"
---

### 算法要点

> 希尔(Shell)排序又称为缩小增量排序，它是一种插入排序。它是对直接插入排序算法的一种优化。

该方法因DL．Shell于1959年提出而得名。

### 算法思想

它的作法不是每次一个元素挨一个元素的比较。而是初期选用大跨步（增量较大）间隔比较，使记录跳跃式接近它的排序位置；
然后增量缩小；最后增量为 1 ，这样记录移动次数大大减少，提高排序效率。

<!--more-->

希尔排序对插入排序一下两点做出改进：

- 插入排序在对几乎已经排好序的数据操作时， 效率高， 即可以达到线性排序的效率
- 但插入排序一般来说是低效的， 因为插入排序每次只能将数据移动一位

![](https://olef5l6y5.qnssl.com/shell-sort-animation.gif)

思路：

- 首先去一个整数d1(d1<n)作为步长，把记录分成d1个分组，所有距离为d1的倍数的记录看成一组，然后在各组组内进行插入排序 
- 然后取d2(d2<d1)  
- 重复上述分组和插入排序操作，直到di=1(i>=1)即所有记录成为一组，最后对这组进行排。一般选d1为n/2、d2=d1/2,...,di=1。

### Golang实现

```
package main

import "fmt"

/**
1 设置gap序列即增量序列，最后一次gap必须是1
2 将相距gap的一组数按照插入排序（注意 插入排序从第二个开始）
3 插入排序 增量为gap 而不是1
*/
func main() {

	array := []int{6, 7, 9, 3, 6, 8, 1, 9, 3}
	n := len(array)

	for gap := n / 2; gap > 0; gap /= 2 {

		for i := gap; i < n; i++ {

			preIndex := i - gap
			current := array[i]
			for ; preIndex >= 0 && array[preIndex] > current; {
				array[preIndex+gap] = array[preIndex]
				preIndex -= gap
			}
			array[preIndex+gap] = current
		}
	}

	fmt.Println(array)
}

```
<a href="https://github.com/sjatsh/algorithms/blob/master/src/github.com/sjatsh/algorithms/sort/shell.go" target="_blank">github地址</a>

### 算法分析

算法性能

| 参数 | 结果 |
|:---|:----|
|排序类型|插入排序|
|排序方法|希尔排序|
|平均时间复杂度|O(Nlog2N)|
|最坏时间复杂度|O(N1.5)|
|最好时间复杂度||
|空间复杂度|O(1)|
|稳定性|不稳定|
|复杂性|较复杂|

#### 时间复杂度

步长的选择是希尔排序的关键，只要最终步长为1都可以。  

希尔排序的增量序列的选择：

- Hibbard序列：{1,3,7……2k-1}，k为大于0的自然数，使用Hibbard序列的希尔排序平均运行时间为θ(n5/4)，最坏情形为O(n3/2)。
- Sedgewick序列：令i为自然数，将9*4-9*2+1的所有结果与4-3*2+1的所有结果进行并集运算，所得数列{1,5,19,41,109……}。使用此序列的希尔排序最坏情形为O(n4/3)，平均情形为O(n7/6)


希尔排序的性能（使用Sedgewick序列）在数据量较大时依然是不错的。如果说插入排序是我们的“初级排序”，用于较少数据或趋于有序数据的情况，那么希尔排序就是我们的“中级排序”，
用于数据量偏多的情况。当然，当数据量极大时，我们将用上我们的“高级排序”——快速排序。