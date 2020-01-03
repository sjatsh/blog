---
layout: post
title: "突破算法之-快速排序"
description: ""
tag: "sort"
date: "2018-08-19 1:30:00 +0800"
categories: "algorithms"
---

### 算法原理

快速排序是图灵奖得主<a href="https://zh.wikipedia.org/wiki/%E6%9D%B1%E5%B0%BC%C2%B7%E9%9C%8D%E7%88%BE" target="_blank">C.R.A. Hoare</a>与1960年提出的一种
划分交换排序，它采用了一种分治策略。  
分治法的基本思想是：将原问题分解为若干个规模更小但结构与原问题相似的子问题。递归地解这些子问题，然后将这些子问题的解组合为原问题的解。  
<!--more-->

利用分治法可将快速排序的分为三步：  

- 在数据集之中，选择一个元素作为”基准”（pivot）
- 所有小于”基准”的元素，都移到”基准”的左边；所有大于”基准”的元素，都移到”基准”的右边。这个操作称为分区 (partition) 操作，分区操作结束后，基准元素所处的位置就是最终排序后它的位置。
- 对”基准”左边和右边的两个子集，不断重复第一步和第二步，直到所有子集只剩下一个元素为止。

![](https://olef5l6y5.qnssl.com/quick_sort.gif)

### Golang实现

```
package main

import "fmt"

/**
1. 从数列中挑出一个元素，称为"基准"（pivot）
2. 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作。
3. 递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序。
*/
func main() {

	// http://jbcdn2.b0.upaiyun.com/2012/01/Visual-and-intuitive-feel-of-7-common-sorting-algorithms.gif
	array := []int{6, 7, 9, 3, 6, 8, 1, 9, 3}
	sort(array, 0, len(array)-1)
	fmt.Println(array)
}

func sort(array []int, left, right int) {
	if left > right {
		return
	}
	var storeIndex = partition(array, left, right)
	sort(array, left, storeIndex-1)
	sort(array, storeIndex+1, right)
}

func partition(array []int, left, right int) int {

	// 数组分区，左小右大
	var storeIndex = left
	// 直接选最右边的元素为基准元素
	var pivot = array[right]

	for i := left; i < right; i++ {
		if array[i] < pivot {
			if i != storeIndex {
				array[storeIndex], array[i] = array[i], array[storeIndex]
			}
			// 交换位置后，storeIndex 自增 1，代表下一个可能要交换的位置
			storeIndex++
		}
	}

	// 将基准元素放置到最后的正确位置上
	array[right], array[storeIndex] = array[storeIndex], array[right]
	return storeIndex
}
```
<a href="https://github.com/sjatsh/algorithms/blob/master/src/github.com/sjatsh/algorithms/sort/quick.go" target="_blank">github地址</a>


