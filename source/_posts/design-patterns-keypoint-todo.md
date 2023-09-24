---
title: Go 实战：设计模式的实现
date: 2023-09-24 21:13:07
categories:
- 八股
tags:
- golang
- 设计模式
---

最近做了很多笔试题，都提到了设计模式。非科班出身的鼠鼠之前了解过这些八股，但是都是浅尝而止。这次试着深入了解一下。用`go`实现相关思路

## 单例模式

单例模式是最常用的一种设计模式，它的设计模式也是很好理解，就是保证返回相同内存的位置的同一实例。创建了就可以反复使用。在这基础上也出现了两种设计理念：

- 饿汉式： 程序启动时就`init()`中创建好实例，不用考虑多线程下并发调用的问题
- 懒汉式： 等需要调用时，在创建实例，需要**考虑多线程的会创建多个实例的异常情况**


所以不难看出难点在于懒汉式的实现。在go中有很优雅的实现方法`sync.Once` 实现单例模式

创建文件
`instance.go` 
```go
package pattern_test

import "sync"

type Instance struct {
	Core string
}

var instance *Instance
var once sync.Once

func GetInstance() *Instance {
	once.Do(func() {
		instance = &Instance{
			Core: "first heart",
		}
	})
	return instance
}

```

`sync.Once` 保证可以保证全局中只使用一次

创建文件
`instance_test.go` 用来测试

```go
package pattern_test

import (
	"fmt"
	"testing"
)

func TestInstance(t *testing.T) {
	instance := GetInstance()

	fmt.Println(instance.Core)
	instance.Core = "second heart"

	instance = GetInstance()
	fmt.Println(instance.Core)
}

```

运行测试符合预期

```txt
=== RUN   TestInstance
first heart
second heart
--- PASS: TestInstance (0.00s)
PASS
ok      go_study/pattern_test   0.248s
```

所以`go`实现懒汉模式还是很方便的