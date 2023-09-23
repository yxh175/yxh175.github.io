---
title: 笔试回顾：美团9-23 
date: 2023-09-23 13:30:50
categories: 
- 笔试
tags:
- 笔试
- 美团
---

## 编程题

总共五道编程题

### 对于一个连串分数数组，计算小于前面最小成绩或者大于前面最大成绩的总数

```go
package main

import (
    "fmt"
)

func main() {
    n := 0
    fmt.Scanln(&n)
    scores := make([]int, n)
    for i := 0; i < n; i++ {
        fmt.Scan(&scores[i])
    }
    cnt := countExameScores(scores)
    fmt.Println(cnt)
}

func countExameScores(scores []int) int {
    n := len(scores)
    if n <= 1 {
        return 0
    }
    minScore := scores[0]
    maxScore := scores[0]
    count := 0
    for i := 1; i < n; i++ {
        if scores[i] < minScore || scores[i] > maxScore {
            count++
        }
        if scores[i] < minScore {
            minScore = scores[i]
        }
        if scores[i] > maxScore {
            maxScore = scores[i]
        }
    }
    return count
}

```

思路比较简单A了

### 2. 给一个24小时制时间的字符串经过多次对分钟的增加和减少，返回最后的时间

```go
package main

import (
    "fmt"
    "strconv"
    "strings"
)

func main() {
    var initTime string
    fmt.Scanln(&initTime)

    var n int
    fmt.Scanln(&n)
    adjustments := make([]string, n)
    adjustNums := make([]int, n)
    // input := bufio.NewScanner(os.Stdin)
    for i := 0; i < n; i++ {
        fmt.Scanln(&adjustments[i], &adjustNums[i])
    }

    fmt.Println(adjustTime(initTime, adjustments, adjustNums))
}

func adjustTime(initTime string, adjustments []string, adjustNums []int) string {
    timeNum := timeToInt(initTime)
    for i := 0; i < len(adjustNums); i++ {
        // 减操作
        if adjustments[i] == "-" {
            timeNum = (timeNum - adjustNums[i]%1440 + 1440) % 1440
        } else if adjustments[i] == "+" {
            timeNum = (timeNum + adjustNums[i]) % 1440
        }
    }
    return intTotime(timeNum)
}

func timeToInt(timeStr string) int {
    parts := strings.Split(timeStr, ":")
    hour, _ := strconv.Atoi(parts[0])
    minute, _ := strconv.Atoi(parts[1])
    return hour*60 + minute
}

func intTotime(minutes int) string {
    hour := minutes / 60
    minute := minutes % 60
    return fmt.Sprintf("%02d:%02d", hour, minute)
}


```

只过了47%，因为没考虑减时间超过1440的情况

### 3. 有一个1 2 1 3 2 1 ... 规律的数组，就是每k进行一次从前到后-1的操作，每一轮k + 1，返回前n个序列的和

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    var n int
    fmt.Scan(&n)
    sqrtN := int(math.Sqrt(float64(2*n))) + 1
    dp := make([]int, sqrtN)
    dp[0] = 1
    for i := 1; i < sqrtN; i++ {
        dp[i] = dp[i-1] + i + 1
    }
    sum := 0
    i := 1
    for n-i >= 0 {
        n -= i
        sum += dp[i-1]
        i++
    }
    // 最后处理
    if n != 0 {
        for j := 0; j < n; j++ {
            sum += i - j
        }
    }

    fmt.Println(sum)
}

```

暴力会超时，需要去找规律

### 4. 好序列， bi = bi-2 给定一个数组，找到最长子序列使其满足好序列

没写出来

### 5. 给一个数组， 连续给q个 l，r，x 的三元组。如果x与从左到右的某个元素的乘积不是完全平方数就返回这个值，否则就返回-1

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    n, q := 0, 0
    fmt.Scanln(&n, &q)
    nums := make([]int, n)
    for i := 0; i < n; i++ {
        fmt.Scan(&nums[i])
    }
    // 去除完全平方因子
    newNums := make([]int, n)
    for i := 0; i < n; i++ {
        newNums[i] = removeSquareFactors(nums[i])
    }
    // fmt.Println(newNums)
    fmt.Scanln()
    for i := 0; i < q; i++ {
        l, r, x := 0, 0, 0
        fmt.Scanln(&l, &r, &x)
        res := findLeft(nums, newNums, l, r, x)
        fmt.Println(res)
    }
}

func findLeft(nums []int, newNums []int, l, r, x int) int {
    flag := isPerfectSquare(x)
    for i := l - 1; i < r; i++ {
        if newNums[i] == x || (flag && newNums[i] == 1) {
            continue
        }
        return nums[i]
    }
    return -1
}

func isPerfectSquare(num int) bool {
    if num < 0 {
        return false
    }

    sqrt := math.Sqrt(float64(num))
    return sqrt == float64(int(sqrt))
}

func removeSquareFactors(n int) int {
    res := n
    for i := 2; i*i <= n; i++ {
        if n%(i*i) == 0 && isPerfectSquare(i*i) {
            res /= i * i
        }
    }
    return res
}

```

这题暴力会超时，没优化前过27%，优化后过0%。考虑可能是`removeSquareFactors`方法写的有问题，需要去修改，应该倒着去除完全平方因子