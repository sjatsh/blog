---
title: "突破算法之-选择排序"
description: ""
tags: "sort"
date: "2016-10-14 14:00:00 +0800"
categories: "algorithms"
---

### 选择排序原理

选择排序思想就是遍历找出最小值或者最大值后维护有序序列，从第一个元素开始依次往后找到最小或者最大元素，然后与第一个元素交换，
然后从第二个元素开始依次往复直到倒数第二个元素找出最小数，时间复杂度O(n^2)。![](https://olef5l6y5.qnssl.com/select_sort.gif) 

<!--more-->

### Golang代码实现

```
package main

import "fmt"

/**
1. 找到数组里最小的元素
2. 让它和数组的第一个元素交换
3. 在剩下元素中找到最小值，与数组的第二个元素交换
4. 如此往复，直到将整个数组排序
*/
func main() {

	var minIndex int
	array := []int{6, 7, 9, 3, 6, 8, 1, 9, 3}

	for i := 0; i < len(array)-1; i++ {
		minIndex = i
		for j := i + 1; j < len(array); j++ {
			if array[j] < array[minIndex] {
				minIndex = j
			}
		}
		array[i], array[minIndex] = array[minIndex], array[i]
	}

	fmt.Println(array)
}

```
<a href="https://github.com/sjatsh/algorithms/blob/master/src/github.com/sjatsh/algorithms/sort/selection.go" target="_blank">github地址</a>

### 选择排序的改进

每次循环时可以分别找出最大值与最小值然后进行双向选择排序，外层判断minIndex==maxIndex+1跳出循环。

```
array := []int{6, 7, 9, 3, 6, 8, 1, 9, 3}
	len := len(array)
	for i := 0; i < len-1; i++ {
		minIndex := i
		maxIndex := len - i - 1
		if minIndex == maxIndex+1 {
			break
		}
		for j := i + 1; j < len; j++ {
			if array[j] < array[minIndex] {
				minIndex = j
			} else if array[j] > array[maxIndex] && j < len-i-1 {
				maxIndex = j
			}
		}
		array[i], array[minIndex] = array[minIndex], array[i]
		array[len-i-1], array[maxIndex] = array[maxIndex], array[len-i-1]
	}

	fmt.Println(array)
```
