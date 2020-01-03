---
title: "突破算法之-直接插入排序"
description: ""
tags: "algorithms"
date: "2018-08-17 14:30:00 +0800"
categories: "algorithms"
---

### 直接插入排序原理

插入排序与选择排序最大的不同在于维护有序列的方式不同： 

- 选择排序通过从无序列中找出最大或最小值放在有序列尾，直到循环到最后一个元素整个数组有序  
- 插入排序是依次把无序列中元素从有序列尾开始比较插入第一个比它小的或者比它大的元素后面，直到循环到最后一个元素整个序列有序

![](https://olef5l6y5.qnssl.com/insertion_sort.gif)  

<!--more-->  

### Golang代码实现

```
package main

import "fmt"

/**
一般来说，插入排序都采用in-place在数组上实现。具体算法描述如下：
1. 从第一个元素开始，该元素可以认为已经被排序；
2. 取出下一个元素，在已经排序的元素序列中从后向前扫描；
3. 如果该元素（已排序）大于新元素，将该元素移到下一位置；
4. 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置；
5. 将新元素插入到该位置后；
6. 重复步骤2~5。
*/
func main() {

	var preIndex, current int
	array := []int{6, 7, 9, 3, 6, 8, 1, 9, 3}

	for i := 1; i < len(array); i++ {

		preIndex = i - 1
		current = array[i]

		for preIndex >= 0 && array[preIndex] > current {
			array[preIndex+1] = array[preIndex]
			preIndex--
		}

		array[preIndex+1] = current
	}

	fmt.Println(array)
}

```
<a href="https://github.com/sjatsh/algorithms/blob/master/src/github.com/sjatsh/algorithms/sort/insertion.go" target="_blank">github地址</a>

