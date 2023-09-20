---
title: 笔试题：途虎养车JAVA开发
date: 2023-09-20 20:42:06
categories:
- 笔试
tags:
- 笔试
- 途虎
---

途虎第三题没写出来，记录一下，复述一下题目大概

给定一些坐标，这些坐标是城市的坐标，(如果每个坐标在一行或者在一列的话则表示两者是可以互相到达的，此时二者只需建立一个工厂)，现在给定一系列坐标，问最少要建多少个工厂

```json
输入
{{0,0}, {0,2}, {2,0}, {1, 1}, {2, 2}}
输出
2
```

刚开始钻牛角尖了，老想着用dfs。实际上是不需要的，因为dfs是为了找到下一个可达点的位置。而这里，坐标全部给你了。只需要你遍历一边。维护一个n*m的表来判断能不能到达

```go
package main

import "fmt"

func main() {
	nums := [][]int{{0, 0}, {0, 2}, {2, 0}, {2, 2}}
	fmt.Println(needFactory(nums))
}

func needFactory(nums [][]int) int {
	rowLen, colLen := 0, 0
	// 找到坐标
	for _, num := range nums {
		rowLen = max(num[0], rowLen)
		colLen = max(num[1], colLen)
	}
	rowLen += 1
	colLen += 1
	visited := make([][]bool, rowLen)
	for i := 0; i < rowLen; i++ {
		visited[i] = make([]bool, colLen)
	}
	cnt := 0
	for _, num := range nums {
		x, y := num[0], num[1]
		if !visited[x][y] {
			cnt++
		}
		for i := x; i < rowLen; i++ {
			visited[i][y] = true
		}
		for i := y; i < colLen; i++ {
			visited[x][i] = true
		}

	}
	return cnt

}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}

```

最终答案和测试样例结果相同



