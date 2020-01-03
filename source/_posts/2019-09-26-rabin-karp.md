---
layout: post
title: "Rabin-Karp在golang中的实现"
date: "2019-09-26 12:01:30 +0800"
tag: "golang"
categories: "golang"
---

### 简介

`Rabin-Karp`字符串快速查找算法和`FNV hash算法`是golang中strings包中字符串查所用到的具体算法，算法的核心就在于循环hash，而 `FNV`则是散列方法的具体算法实现。

### 算法思想

**Rabin-Karp算法思想：**

- 假设待匹配字符串长度M，目标字符串长度N（N>M）
- 首先计算待匹配字符串hash，计算目标字符串前M个字符hash
- 比较前两个hash值，比较次数N-M+1
  - 若hash不相等，继续计算目标字符串下一个长度为M的hash并继续循环比较
  - 若hash相等则再次判断字符串是否相等已确保正确性



**FNV hash：** 

将字符串看作是字符串长度的整数，这个数的进制是一个质数。计算出来结果之后，按照哈希的范围求余数得到结果。

其中不同机制对应质数分别是：

```shell
32 bit FNV_prime = 224 + 28 + 0x93 = 16777619

64 bit FNV_prime = 240 + 28 + 0xb3 = 1099511628211

128 bit FNV_prime = 288 + 28 + 0x3b = 309485009821345068724781371

256 bit FNV_prime = 2168 + 28 + 0x63 = 374144419156711147060143317175368453031918731002211

512 bit FNV_prime = 2344 + 28 + 0x57 =
35835915874844867368919076489095108449946327955754392558399825615420669938882575
126094039892345713852759

1024 bit FNV_prime = 2680 + 28 + 0x8d =
50164565101131186554345988110352789550307653454047907443030175238311120551081474
51509157692220295382716162651878526895249385292291816524375083746691371804094271
873160484737966720260389217684476157468082573
```

以上这几个数都是质数（哈希的理论基石，质数分辨定理)，简单地说就是：n个不同的质数可以“分辨”的连续整数的个数和他们的乘积相等。“分辨”就是指这些连续的整数不可能有完全相同的余数序列。[证明详见](http://wenku.baidu.com/view/16b2c7abd1f34693daef3e58.html)

如果想要得到不是上面进制的hash：  

比如想得到24位的哈希值，方法：取上面比24大的最小的位数，当然是32了，先算对应32位哈希值，再转换成24位的。  
转换方法：32 - 24 = 8， 好了把得到的32砍成两段，高8位最和低24位。第8位与低24位中的低8位做抑或，得到的24位值是最终结果。
（**hash**>>24) ^ (**hash** & 0xFFFFFF);  

如果想得到范围在0~9999的哈希值，方法：取上面比9999大的最小的位数，当然是32，先算对应32位哈希值，再mod（9999 +１）。  

如上所述，结合`Rabin-Karp`的思想加上`FNV hash`就可以实现所谓的字符串快速查找算法了。



### 结合golang源码

`src/strings/strings.go`

```
// Rabin-Karp 中需要使用的32位FNV hash算法中的基础质数（相当于进制）
const  primeRK = 16777619

// hash散列方法， 返回字符串hash以及 primeRK的k-1（len(sep)-1）次方  
func hashStr(sep string) (uint32, uint32) {  
	hash := uint32(0)  
	for i := 0; i < len(sep); i++ {  
		hash = hash*primeRK + uint32(sep[i]) // 循环得到字符串hash  
	}  
     
    // 位运算巧妙的获取 primeRK 的 len(sqp)-1 次方  
	var pow, sq uint32 = 1, primeRK  
	for i := len(sep); i > 0; i >>= 1 {  
		if i&1 != 0 {  
			pow *= sq  
		}  
		sq *= sq  
	}
	return hash, pow  
}  
  
func indexRabinKarp(s, substr string) int {  
	// Rabin-Karp search  
	hashss, pow := hashStr(substr)  
	n := len(substr)  
	var h uint32  
    // 计算目标字符串前n位hash并与待匹配字符串hash进行对比  
	for i := 0; i < n; i++ {  
		h = h*primeRK + uint32(s[i])  
	}  
    // hash相同并且字符串相等则返回当前位置下标  
	if h == hashss && s[:n] == substr {  
		return 0  
	}  
  
    // Rabin-Karp 算法的精华所在，相面详细介绍  
	for i := n; i < len(s); {  
		h *= primeRK  
		h += uint32(s[i])  
		h -= pow * uint32(s[i-n])  
		i++  
		if h == hashss && s[i-n:i] == substr {  
			return i - n  
		}  
	}  
	return -1  
}
```



结合源码可以知道：如果现在我们要求第i位往后k个长度字符串的hash可以列个公式

![](/img/fnv_hash_hi.png)

其中：s[i] 表示第i位字节对应32位整数也就是上面`uint32(s[i])` （这里强转一下也就是对2^32次方取余了），R 就是 对应进制的`FNV_prime`。

由上述类推H(i+1)的hash公式就是：

![](/img/fnv_hash_hi1.png)

由此可以看出来：每次我们其实不用重新计算整个字符串的hash而是直接原hash值乘以R加上s[k-1]并且减去s[i]R^(k-1)，这里也就是 FNV_prime的k-1次方，对应上面代码：

```
var pow, sq uint32 = 1, primeRK  
for i := len(sep); i > 0; i >>= 1 {
	if i&1 != 0 {
		pow *= sq  
	}  
	sq *= sq
}
```

相对于暴力匹配O(mn)是时间复杂度， Rabin-Karp 的时间复杂度在O(m+n)， 最坏的情况每次hash相同字符串不相同时间复杂度会变成O(mn)但是这种情况比较罕见。 



 Rabin-Karp 还有个优点在于他可以进行多模式匹配，比如论文重复性检测，只要预热计算出所有带匹配字符串的hash，目标字符串的遍历比较时只是多一步比较所有待匹配字符串hash。如果待匹配字符串个数是k，那么 Rabin-Karp 的时间复杂度是O(nk)。