---
title: 面试经验：金山办公二面
date: 2023-10-18 18:37:08
categories:
- 面经
tags:
- 面经
---

记录一下2023年10月18日金山办公二面，总体体验良好，有一些没答上来，继被苏小妍语言层面冲击后，又被冲击了一次。水平还是有很大的差距啊

## 自我介绍

简单的进行一个自我介绍

## GO语言场景题

这一块我以为当时苏小妍问过之后，恶补之后能够坦然应对，没想到还是被拷打了。

### new 和 make的区别

- new 返回一个指向该类型内存地址的指针
- slice 返回的还是该类型

### 前端传输一个表单，需要对这个表单的字段检查一下，判断字段名称是否超长，怎么检查？

刚开始有点不理解这个说法，和面试官梳理了一下。感觉是有坑的。

### 写代码中是传指针还是传值好？

两者都各有优缺点

不能说某一个一定是好的。

如果值量小，且不希望修改值，推荐使用值传递，减小GC得负担，如果是大值，如结构体等较大对象，可以优先考虑去传指针

一般情况下，对于需要修改原对象值，或占用内存比较大的结构体，选择传指针。对于只读的占用内存较小的结构体，直接传值能够获得更好的性能。

### defer 是在return之前还是之后？

在return 之后

### 多个defer会怎么样？

因为defer 是一种先进后出得栈存储结构，它会把对应得函数指针在执行一边

### defer函数的值，会被之后影响吗？

如果是值传递得话是不会影响，如果是闭包调用和引用传递的话是会影响的

```go

func TestDeferResChange(t *testing.T) {
	deferChange1()
	deferChange2()
	deferChange3()
}

func deferChange1() {
	i := 0
	defer func() {
		fmt.Println(i)
	}()
	i++
}

func deferChange2() {
	i := 0
	defer fmt.Println(i)
	i++
}

func deferChange3() {
	i := 0
	defer func(x *int) {
		fmt.Println(*x)
	}(&i)
	i++
}

// 测试结果
// === RUN   TestDeferResChange
// 1
// 0
// 1
// --- PASS: TestDeferResChange (0.00s)
// PASS
```

### defer之间执行多个函数是什么顺序？

defer 相当于把函数指针压入栈中，结束后再将defer中的函数依次去执行，所以是按顺序的

```go
func TestDeferRes(t *testing.T) {
	deferRes()
}

type as struct {
}

type bs struct {
}

type cs struct {
}

func deferRes() {
	as := as{}
	defer as.a().b().c()
}
func (a as) a() bs {
	fmt.Println("a")
	return bs{}
}
func (b bs) b() cs {
	fmt.Println("b")
	return cs{}
}
func (c cs) c() {
	fmt.Println("c")
}

// 输出结果

// === RUN   TestDeferRes
// a
// b
// c
// --- PASS: TestDeferRes (0.00s)
// PASS
```

### 为什么说go是天然高并发？

- goroutine  `go`
- channel  `chan`
- 调度器  `GMP`

go语言在设计的时候从关键字层面实现了多协程开发，好像语言天生支持高并发一样。

### 内存泄漏是什么？

内存泄漏是指GC无法清除的内存，一般来说会发生再内存中某个函数出现阻塞的情况，或者持续的占用导致无法收回

### 之前有没有遇到过，要怎么解决？

可以通过go tool pprof分析工具去解决，具体流程如下

## 八股及业务场景

### GMP讲一下？

### 如果一个协程进来之后它是怎么处理的？

### 什么是跨域？

### http 和 https 算跨域吗？

### 跨域问题怎么解决呢？

### 协作文档怎么实现的？

### docker和k8s是什么？

### docker和虚拟机的区别？

### dockerfile 有没有写过？

### 有没有用过k8s？

### k8s内部是怎么通信的？

### 假设有个服务请求进来，这个调用流程是什么？

### target_node、节点是什么？

### git 一般怎么用？

### rebase呢， 有了解过是干什么的

### 说一下url的键入到浏览器的流程？

### dns中A、B、C类地址

### Mysql有哪些数据引擎

### 有哪些锁？

### 主键索引和唯一索引的区别？

### mysql慢查询有了解过吗？

### mysql慢查询怎么优化？

### 怎么知道索引有没有生效？

### 有没有用过MemoryCache？

### redis有没有用过，怎么实现一个redis分布式锁？

### redis 有哪些数据类型?

## 反问

### 部门是怎么分配的？

### 技术栈问题？
