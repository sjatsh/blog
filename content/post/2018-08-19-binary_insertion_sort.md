---
layout:  post
title:   "突破算法之-二分插入排序"
date:    "2018-08-19 00:00:00 +0800"
categories: "技术"
---

### 二分插入排序原理

二分法是对直接插入的改进，由直接插入排序的循环遍历找出插入点改为二分法找出插入点，时间复杂度O(nlogn)。  
<!--more-->
二分法插入排序是在插入第i个元素时，对前面的0～i-1元素进行折半，先跟他们中间的那个元素比，如果小，则对前半再进行折半，
否则对后半进行折半，直到left>right，然后再把第i个元素前1位与目标位置之间的所有元素后移，再把第i个元素放在目标位置上。
![](https://olef5l6y5.qnssl.com/binary_sort.gif)

### Golang实现

```
package main

import "fmt"

// 二分插入排序
// 算法思想简单描述：
//在插入第i个元素时，对前面的0～i-1元素进行折半，先跟他们
//中间的那个元素比，如果小，则对前半再进行折半，否则对后半
//进行折半，直到left>right，然后再把第i个元素前1位与目标位置之间
//的所有元素后移，再把第i个元素放在目标位置上。
func main() {

	array := []int{6, 7, 9, 3, 6, 8, 1, 9, 3}

	var start, end, temp, mid int

	for i := 1; i < len(array); i++ {

		start = 0
		end = i - 1
		temp = array[i]

		for start <= end {
			mid = (start + end) / 2
			if temp < array[mid] {
				end = mid - 1
			} else {
				start = mid + 1
			}
		}
		//循环完后，start=end+1,此时start为当前插入数字所待坑位
		//把坑位给当前插入的数据挪出来
		for j := i - 1; j >= start; j-- {
			array[j+1] = array[j]
		}
		//将当前插入数字挪入它该待的坑位
		array[start] = temp
	}

	fmt.Println(array)
}
```
<a href="https://github.com/sjatsh/algorithms/blob/master/src/github.com/sjatsh/algorithms/sort/binaryInsertion.go" target="_blank">github地址</a>