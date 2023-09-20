---
title: GO：quick-sort实现
date: 2023-09-17 09:54:38
categories:
- 算法
tags: 
- 快排
- 排序算法
---
# 快排


## 快排思想
快速排序: 每次排序会定位出一个位置，左右两边在分别排序。思路大致是这样的。但是不知道网上很多博客里用go实现的都很奇怪。所以决定自己捋捋。强化一下记忆

![快排思路](https://img1.imgtp.com/2023/09/17/Rbq8rX3b.png)

为了方便起见，每次都取最右边的进行排序（这样不用额外的判断是否是key值）。哨兵元素的选取可能会导致排序退化成冒泡排序

## 代码演示

代码如下

```go
package main

import "fmt"

func main() {
	s := []int{2, 3, 1, 5, 6, 8}
	quickSort(s, 0, len(s)-1)
	fmt.Println(s)
}

func quickSort(nums []int, l, r int) {
	if l < r {
		pivot := partition(nums, l, r)
		quickSort(nums, l, pivot-1)
		quickSort(nums, pivot+1, r)
	}
}

func partition(nums []int, l, r int) int {
	key := nums[r]
	// i 左边表示小于key， i右边表示大于key
	i := l

	// 通过j进行遍历
	for j := l; j < r; j++ {
		if nums[j] < key {
			nums[i], nums[j] = nums[j], nums[i]
			i++
		}
	}
	// 最后i的位置即为key所在的位置
	nums[i], nums[r] = nums[r], nums[i]
	return i
}

```

如果理解了，还可以尝试把`partition`也塞到`quickSort`中，去减少栈内存的使用的压力

`partition`是定位`key`的函数。通过i和j两个指针去查询

- `i`的作用: `i`之前的所有数都小于`key`，不包括`i`
- `j`的作用：单纯用来进行遍历，查到小于`key`的要放到`i`的位置，并让`i++`


`quickSort`中的`l<r`的作用是递归出口也表示某个分支排序的结束。知道的key就不用参与排序了

## 复杂度分析

快排是原地排序算法, 因为要设计到哨兵元素的选取。如果有和哨兵元素相同的值。不一定能保证原有的顺序，所以是不稳定排序

评价标准 | 平均| 最坏
---| ---| ---|
时间复杂度|o(nlogn)|o(n^2)
空间复杂度|o(logn)|o(n)
