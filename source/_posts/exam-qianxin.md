---
title: 笔试记录：奇安信
date: 2023-10-09 15:53:38
cover: https://s2.loli.net/2023/10/09/ZV7ls5CPmbKwYJI.png
categories:
- 笔试
tags:
- 笔试
---

2023年10月9日笔试，整体难度不高，40min结束战斗，不太记得清除题了，只知道是什么类型的，简单的描述一下

## 芭莎慈善夜明星捐款

核心代码模式
实际就是![打家劫舍2](https://leetcode.cn/problems/house-robber-ii/)，明星围坐在一起进行捐款

```go
func rob(nums []int) int {
    if len(nums) == 1 {
        return nums[0]
    }
    if len(nums) == 2 {
        return max(nums[0], nums[1])
    }
    return max(maxSteal(nums[1:]), maxSteal(nums[:len(nums)-1]))
}

func maxSteal(nums []int) int {
    a := nums[0]
    b := max(nums[0], nums[1])
    c := b
    for i := 2; i < len(nums); i ++ {
        c = max(a + nums[i], b)
        a = b
        b = c
    }
    return c
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

## 判断资源使用是否异常

题目巨长，没仔细看，总的来说就是给定一个数组, 然后1是获取资源，0是损失资源，最终满足两个要求

- 资源在中间不会为负
- 资源最后为0

```text
1 0 1 1 1 0 0 0

true

1 0 1 1 1 0 0 1

false

1 0 1 1 0 0 0 1

false
```

本地没写直接线上写的, 思路比较简单一个cnt做记录

```go
func isGood(nums []int) bool {
    cnt := 0
    for i := 0; i < len(nums); i ++ {
        if nums(i) == 1 {
            cnt ++
        } else {
            cnt --
        }
        if cnt < 0 {
            return false
        }
    }
    return cnt == 0
}

```

就这么一小点，感觉应该是没HC了。
