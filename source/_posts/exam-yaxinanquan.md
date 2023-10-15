---
title: 笔试记录：亚信安全
date: 2023-10-15 20:26:14
tags:
- 笔试
---

## 把小端字节序的IPv4地址转换成大端字节序表示，并用点分法输出

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	var input string
	fmt.Scanln(&input)
	n := len(input)
	nums1 := hexToInt(input[n-2:])
	nums2 := hexToInt(input[n-4 : n-2])
	nums3 := hexToInt(input[n-6 : n-4])
	nums4 := hexToInt(input[n-8 : n-6])
	fmt.Printf("%v.%v.%v.%v", nums1, nums2, nums3, nums4)
}

func hexToInt(hex string) int {
	num, _ := strconv.ParseInt(hex, 16, 0)
	return int(num)
}

```


## 验证括号组成的字符串是否正常

leetcode 原题：[有效括号](https://leetcode.cn/problems/valid-parentheses/)

```go
package main

import (
	"fmt"
)

func main() {
	var input string
	fmt.Scanln(&input)
	fmt.Println(isValid(input))
}

func isValid(s string) bool {
	n := len(s)
	if n%2 != 0 {
		return false
	}

	set := map[byte]byte{
		')': '(',
		']': '[',
		'}': '{',
	}

	stack := []byte{}
	for i := 0; i < n; i++ {
		if set[s[i]] > 0 {
			if len(stack) == 0 || stack[len(stack)-1] != set[s[i]] {
				return false
			}
			stack = stack[:len(stack)-1]
		} else {
			stack = append(stack, s[i])
		}
	}
	return len(stack) == 0
}

```

## 求两个ip地址的最大相同前缀

输入

```text
192.168.1.1 192.168.1.3
```

输出

```text
Prefix length:  30
```

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
)

func main() {
	var ip1, ip2 string
	fmt.Scanln(&ip1, &ip2)
	ip1Byte := ipTobit(ip1)
	ip2Byte := ipTobit(ip2)
	sum := 0
	for i := 0; i < len(ip1Byte); i++ {
		if ip1Byte[i] != ip2Byte[i] {
			break
		}
		sum++
	}
	fmt.Println("Prefix length:", sum)
}

func ipTobit(ip string) []byte {
	res := make([]byte, 32)
	ipStr := strings.Split(ip, ".")
	ipNum := make([]int, len(ipStr))
	for i := 0; i < len(ipStr); i++ {
		ipNum[i], _ = strconv.Atoi(ipStr[i])
	}
	for i := 0; i < len(ipNum); i++ {
		binaryStr := intTobin(ipNum[i])
		for j := 0; j < len(binaryStr); j++ {
			res[i*8+j] = binaryStr[j]
		}
	}
	return res

}

func intTobin(num int) string {
	binaryStr := strconv.FormatInt(int64(num), 2)

	for len(binaryStr) < 8 {
		binaryStr = "0" + binaryStr
	}
	return binaryStr
}

```