---
title: GO：如何实现一个堆
date: 2023-09-19 09:10:36
tags:
- Golang
- 堆
- 堆排序
---

一些leetcode算法题中经常用到堆排序去解决问题比如常见

[数组第K大个数](https://leetcode.cn/problems/kth-largest-element-in-an-array/)
[658. 找到 K 个最接近的元素](https://leetcode.cn/problems/find-k-closest-elements/)
Golang 给出了主要的实现堆队列的主要方法，但是由于堆的数据结构都可以使用，所以需要自己去实现一些必要的接口，才能正常使用一个堆

## 堆的接口

`heap.Interface` 接口源码

```go
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}
```

其中`sort.Interface` 接口源码

```go
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

也就是说要实现一个堆，至少要实现以上的接口。所以我们要想使用heap定义的堆，就需要实现这五个方法。以下简单计算一个矩形面积的堆排序

```go
package main

import (
	"container/heap"
	"fmt"
)

type Rectangle struct {
	width  int
	height int
}

func (rec *Rectangle) Area() int {
	return rec.height * rec.width
}

type RectHeap []Rectangle

func (hp RectHeap) Len() int {
	return len(hp)
}

// 按从小到大排列
func (hp RectHeap) Less(i, j int) bool {
	return hp[i].Area() < hp[j].Area()
}

func (hp RectHeap) Swap(i, j int) {
	hp[i], hp[j] = hp[j], hp[i]
}

func (hp *RectHeap) Push(h interface{}) {
	*hp = append(*hp, h.(Rectangle))
}

func (hp *RectHeap) Pop() (x interface{}) {
	n := len(*hp)
	x = (*hp)[n-1]
	*hp = (*hp)[:n-1]
	return x
}

func main() {
	hp := &RectHeap{}
	for i := 2; i < 6; i++ {
		*hp = append(*hp, Rectangle{i, i})
	}

	fmt.Println("原始slice: ", hp)

	// 堆操作
	heap.Init(hp)
	heap.Push(hp, Rectangle{100, 10})
	fmt.Println("top元素：", (*hp)[0])
	fmt.Println("删除并返回最后一个：", heap.Pop(hp)) // 最后 一个元素
	fmt.Println("最终slice: ", hp)
}

```

通过实现五个接口需要实现的函数，即可完成要求



