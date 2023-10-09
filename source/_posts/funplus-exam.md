---
title: FunPlus笔试题
date: 2023-09-15 20:37:18
categories:
  - 笔试
tags: 
- 笔试 
- funPlus
---

## FunPlus笔试题

记录一下题目和解答

## 第一题

趣小加非常喜爱玩扑克，现在需要从若干副扑克牌中随机抽5张牌，判断是不是一个顺子，即这5张牌是不是连续的。2~10为数字本身，A为1，J为11，Q为12，K为13；而大、小王为0，可以看成任意牌。另外，“10， J(11), Q(12), K(13), A(1)”也是顺子，请你帮助趣小加实现上述功能

```go
package main

import (
	"fmt"
	"sort"
)

func isStraight(nums []int) bool {
	// 把A当1时
	joker := 0
	sort.Ints(nums)
	for i := 0; i < 4; i++ {
		if nums[i] == 0 {
			joker++
		} else if nums[i] == nums[i+1] {
			return false
		}
	}
	flag := nums[4]-nums[joker] < 5
	joker = 0
	for i := 0; i < 4; i++ {
		if nums[i] == 1 {
			nums[i] = 14
		}
	}
	sort.Ints(nums)
	for i := 0; i < 4; i++ {
		if nums[i] == 0 {
			joker++
		} else if nums[i] == nums[i+1] {
			return false
		}
	}
	return nums[4]-nums[joker] < 5 || flag
}

func main() {
	// 测试用例
	cards1 := []int{1, 2, 3, 4, 5}   // 是顺子
	cards2 := []int{0, 1, 2, 4, 5}   // 是顺子，0作为3
	cards3 := []int{0, 0, 1, 2, 6}   // 不是顺子，0作为3和4
	cards4 := []int{1, 2, 3, 4, 4}   // 不是顺子，有重复的牌
	cards5 := []int{0, 0, 1, 10, 11} // 顺子 0 作为12和13

	fmt.Println(isStraight(cards1)) // 输出 true
	fmt.Println(isStraight(cards2)) // 输出 true
	fmt.Println(isStraight(cards3)) // 输出 false
	fmt.Println(isStraight(cards4)) // 输出 false
	fmt.Println(isStraight(cards5)) // 输出 true
}

```

本地测试了一下发现结果是正确的。
