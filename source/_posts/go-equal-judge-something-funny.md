---
title: GO底层：薛定谔的指针
date: 2023-10-28 16:18:21
cover: https://s2.loli.net/2023/10/28/WNqXGb82tSnl7yA.png
top_img: https://s2.loli.net/2023/10/09/Y2EknVqJy5fWgpr.png
categories: 
- GO
tags:
- 底层剖析
- 指针
---

## 问题出现

今天在`Golang`交流群划水，突然看到一个哥们提出了这么一个问题

![奇怪的返回](https://s2.loli.net/2023/10/28/ZaoFyBdmIHufsl8.png)

嗯？这又是Go的什么坑，让俺瞅一瞅，先简单的复现一下

```go
package main

import "fmt"

type A struct{}

func main() {
	a := &A{}
	b := &A{}
	c := &A{}

	fmt.Println(a == b, b == c)
	fmt.Println(a, b)
}

// true false
// &{} &{}
```

不带`fmt.Println(a, b)`

```go
package main

import "fmt"

type A struct{}

func main() {
	a := &A{}
	b := &A{}
	c := &A{}

	fmt.Println(a == b, b == c)
	// fmt.Println(a, b)
}
// false false
```

好家伙，这瞬间让我想到了薛定谔的猫。不观测它，它的状态是未知的，观测的一瞬间状态坍缩到一个确定的值 🐂。还好这种事情是在我们程序发生，想观测这里的原因还不是手到擒来。

## 剖析

其实看到这个问题，有一定`Go`底层基础的肯定上来猜个大概，这跟内存逃逸肯定没跑了。

### 验证思路

猜测是空结构体的指针在`GC`的优化导致的异常。而这正是因为我们调用Println观测导致的。既然不能用fmt包去看，要观测这个问题，其实还可以用雪藏的`println`去解决, `println`是`Go`在实现自举的时候供开发人员打印使用的，后续并不能保证其能正常工作。如果想了解和`fmt`包的不同，可以看看这个知乎问题[Golang中fmt.Println和直接println有什么区别？](https://www.zhihu.com/question/335186436)

所以我们改一下代码看看

```go
package main

import "fmt"

type A struct{}

func main() {
	a := &A{}
	b := &A{}
	c := &A{}
	d := &A{}

	println("a, b的地址： ", a, b)
	fmt.Println(b == a)
	fmt.Printf("c, d 的地址：%p\t%p\t%t\n", c, d, c == d)
}
// a, b的地址：  0xc00007df2f 0xc00007df2f
// false
// c, d 的地址：0x5d9540	0x5d9540	true
```

明明是一起创建，为什么a,b的地址是一类，但是又是`false`，c、d的地址又是一类，两者又是true。奇怪奇怪。这时候我下意识的想到。既然后者的`fmt.Println()`中的调用导致了，指针不同，有没有可能是编译器的问题。考虑到`fmt`中大量的反射相关方法的调用。编译器优化的**逃逸分析**，然后`println`是栈上指针的比较，`fmt.Println`是堆上指针的比较。这多半没跑了，剩下的就是验证了。

### 逃逸分析

如果不理解逃逸分析是什么，可以先看一下这篇文章[逃逸分析](https://geektutu.com/post/hpg-escape-analysis.html)。简单来讲，编译器决定内存分配位置的方式，就称之为逃逸分析(escape analysis)。逃逸分析由编译器完成，作用于编译阶段。

我们试着使用go的逃逸分析指令去观测一下。

```shell
go run -gcflags="-m -l" main.go
```

得到这样的结果

```text
# command-line-arguments
.\main.go:8:7: &A{} does not escape
.\main.go:9:7: &A{} does not escape
.\main.go:10:7: &A{} escapes to heap
.\main.go:11:7: &A{} escapes to heap
.\main.go:14:13: ... argument does not escape
.\main.go:14:16: b == a escapes to heap
.\main.go:15:12: ... argument does not escape
.\main.go:15:54: c == d escapes to heap
a, b的地址：  0xc000109f2f 0xc000109f2f
false
c, d 的地址：0xe89560   0xe89560        true
```

可以看到8、9 两行的创建没有逃逸，10、11则逃逸到堆上了。那么两者出现区别的原因看来是找到了。那为什么两者的值又不相同呢？

### 为什么逃逸后相等

这个主要和`go`的`gc`优化有关，对于一个空结构体，`go`是如何优化的大家可以看这篇文章的详细部分[Go 最细节篇 — 空结构体是什么?](https://juejin.cn/post/6908733156707287048)。我简单取其中重要的部分

![内存管理特殊处理](https://s2.loli.net/2023/10/28/2AjHfwv1xzSFhJK.png)

所以空`struct`在逃逸后本质上指向了`zerobase`，其两者比较就是相等的，返回了`true`。

### 为什么不逃逸不相等

这里其实我也并不理解，因为我们看到输出，两个a、b两个指针实际上值是相同的，但是仍然是`false`。这其中肯定有其他原因。查看`golang`的[语言规范](https://go.dev/ref/spec#Comparison_operators)中可以看到这样一段描述

![go语言规范](https://s2.loli.net/2023/10/28/u7XRlBqSGgNyoi8.png)

注意这句话`Pointer types are comparable. Two pointer values are equal if they point to the same variable or if both have value . Pointers to distinct zero-size variables may or may not be equal. nil`

也就是说在没逃逸的场景下，两个空`struct`的比较动作，并不是真的在比较。实际上已经在代码优化阶段被直接优化掉，转为了`false`。所以`a==b`两者实际没有进行比较，直接返回的false
