---
title: 笔试记录：集度
date: 2023-10-09 21:03:15
cover: https://s2.loli.net/2023/10/09/zkQG12UoBTr5lYO.png
tags:
- 笔试
---

集度笔试，编程题全考字符串了，难度不高

## 存在一个子序列数组和一个字符串，判断多个子序列是否存在于字符串中，是的话返回对应的子序列数组。

有一些要求

- 按子序列数组顺序返回
- 重复不返回

只能过80%，代码如下

```go

package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
)

func main() {
    input := bufio.NewScanner(os.Stdin)
    input.Scan()
    dataStr := strings.Split(input.Text(), ",")
    input.Scan()
    s := input.Text()

    res := parseWord(s, dataStr)
    for i := 0; i < len(res); i ++ {
        if i != 0 {
            fmt.Print(",")
        }
        fmt.Print(res[i])
    }
}

func parseWord(s string, subsequence []string) []string {
    result := []string{}
    set := map[string]bool{}
    for _, sub := range subsequence {
        if !set[sub] && isSub(s, sub) {
            result = append(result, sub)
            set[sub] = true
        }
    }
    return result
}

func isSub(s, sub string) bool {
    i, j := 0, 0
    for i < len(s) && j < len(sub) {
        if s[i] == sub[j] {
            j++
        }
        i++
    }

    return j == len(sub)
}

```

## 重复的DNA序列

leetcode原题：[重复的DNA序列](https://leetcode.cn/problems/repeated-dna-sequences/)

简单起见直接用hash表写了，核心代码模式

```go
func findRepeatedDnaSequences(s string) (ans []string) {
    cnt := map[string]int{}
    for i := 0; i <= len(s)-10; i++ {
        sub := s[i : i+10]
        cnt[sub]++
        if cnt[sub] == 2 {
            ans = append(ans, sub)
        }
    }
    return
}
```
