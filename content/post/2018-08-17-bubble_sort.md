---
layout: post
title: "突破算法之-冒泡排序"
date: "2018-08-17 13:30:00 +0800"
categories: "技术"
---

### 冒泡排序原理

对数组内所有n个元素依次进行比较和交换位置，较大的元素往上浮较小的元素往下沉，外层进行n-1次循环后数组会从小到大排好序，
时间复杂度O(n^2)。![](https://olef5l6y5.qnssl.com/bubble_sort.gif) 
<!--more-->

### Golang代码实现

```
package main

import "fmt"

/**
1. 比较相邻的元素。如果第一个比第二个大，就交换它们两个；
2. 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对，这样在最后的元素应该会是最大的数；
3. 针对所有的元素重复以上的步骤，除了最后一个；
4. 重复步骤1~3，直到排序完成。
*/
func main() {
	array := []int{6, 7, 9, 3, 6, 8, 1, 9, 3}
	len := len(array)
	for i := 0; i < len-1; i++ {
		for j := i+1; j < len; j++ {
			if array[i] > array[j] {
				array[i], array[j] = array[j], array[i]
			}
		}
	}
	fmt.Println(array)
}
```
<a href="https://github.com/sjatsh/algorithms/blob/master/src/github.com/sjatsh/algorithms/sort/bubble.go" target="_blank"">github地址</a>




