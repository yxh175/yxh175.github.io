---
title: Go：recover底层及使用
date: 2023-10-21 21:47:42
categories:
- Golang
tags:
- Golang
---

`panic`，基本学习golang语言必须要了解的一个知识，同时还有一同存在的recover

- panic： 能够改变程序的控制流，通过调用panic后会立刻停止执行当前函数的剩余代码，并在当前Goroutine中递归执行调用方的defer
- recover：中止panic造成的程序崩溃，该函数只能在defer中发挥作用，在其他作用域中不会发挥作用

## 最佳实践

### 使用recover捕捉panic

```go
package main

import "fmt"

func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("recover 捕获成功", err)
		}
	}()

	panic("panic err")
}
```
