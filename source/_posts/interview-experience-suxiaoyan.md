---
title: 面试经验：苏小妍
date: 2023-10-17 16:20:10
categories: 
- 面经
tags:
- 面经
---

2023年10月17日golang开发工程师 苏小妍一面

**注意**：一定要检查自己的腾讯会议的名字，不要是自己的本人的名字。否则真的很尴尬。甚至有可能因此挂掉。

总共有两个面试官，一个主面试官，一个副面试官。

主面试官问的语言层面（很偏、感觉大受震撼），副面试官问的是实习与其他八股相关

## 主面试官

### slice 中nil和空的区别？

- slice是nil的时候是没有指向任何实例的
- slice为空的时候，是可以存在slice的，只不过这个slice里面是空的

### 一个nil的slice的在序列化会出现什么？

一个nil的slice在序列化的时候理论上来说会出现空指针异常，不多说测一测

```go
func TestSliceJson(t *testing.T) {
	var sNil []int

	fmt.Println("sNil is", sNil == nil)

	// res是[]byte类型
	res, err := json.Marshal(sNil)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(string(res))
}

// 运行结果
// === RUN   TestSliceJson
// sNil is true
// null
// --- PASS: TestSliceJson (0.00s)
// PASS
```

发现输出结果是null，绷不住了我回答的是空指针异常

### slice 遍历时，append会出现死循环吗？

当使用range遍历时和使用`for i := 0; i < len(s); i ++` 

```go
func TestSliceAppend(t *testing.T) {
	s := []int{1, 2, 3}

	// range
	for i, v := range s {
		fmt.Println(v)
		s = append(s, i)
	}

	// 普通for循环
	for i := 0; i < len(s) && i < 100; i++ {
		s = append(s, i)
		fmt.Println(s[i])
	}
}

// 输出
// range
// 1
// 2
// 3
// 普通for循环
// 1
// 2
// 3
// 0
// 1
// 2
// 0
// 1
// 2
// 3
// 4
// 5
// 6
// 7
// 8
// 9
// 10
```

可以看到range是不会出现死循环，但是普通的for循环就会死循环。

机理上不同的原因是：

- range range相当于是快照，一种值拷贝所以不会感知到append的结果
- for 相当于是对i的控制，每一次会进行一种对比

### map是并发安全的吗？

不是，为了解决这种并发安全问题，通常使用`sync.Map`或者使用`sync.RWMutex`进行锁之间的冲突解决

### sync.Map底层原理？

给自己挖坑了，没答上来

百度一下这篇文章写的很详细[Sync.Map底层](https://segmentfault.com/a/1190000043764275)

```go
type Map struct {
   mu Mutex // 锁，保证写操作及dirty晋升为read的线程安全

   read atomic.Value // readOnly 只读map

   dirty map[any]*entry // 脏map，当内部有数据时就一定包含read中的数据

   misses int // read未命中次数，当达到一定次数时会触发dirty中的护具晋升到read
}

```

可以看到主要的思想是一个锁，一个`read map`和`dirty map`，当内部有数据时，一定包含read中的数据。
`

### dirtyMap是什么(sync.Map的底层)？

底层没看T-T

### defer 会修改return的返回值吗

跟面试官说了一下整体的顺序，如果是一个变量，是会修改返回的值

```go
func TestXxx(t *testing.T) {

	fmt.Println("test1: ", deferTest1())
	fmt.Println("test2: ", deferTest2())
	fmt.Println("test3: ", deferTest3())
	fmt.Println("test4: ", deferTest4())
}

func deferTest1() (x int) {
	x = 0
	defer func() {
		x = 100
	}()
	return x
}

func deferTest2() int {
	x := 0
	defer func() {
		x = 100
	}()
	return x
}

func deferTest3() int {
	x := 0
	defer func(r int) {
		r = 100
	}(x)
	return x
}

func deferTest4() int {
	x := 0
	defer func(r *int) {
		*r = 100
	}(&x)
	return x
}

// 输出结果

// test1:  100
// test2:  0
// test3:  0
// test4:  0
```

之后大概测试了一下，发现只有在返回值是在刚开始就确定好的话，就会导致返回结果被修改

## 副面试官

### 链表反转如何实现（简述算法思路）？

pre 记录前节点
now 记录当前节点
nxt 记录下一个节点

依次前推

### 挖实习

### 问docker和k8s是什么，他们之间的关系？

Docker和K8S之间是一种协同的关系，Docker负责应用级的容器构建、打包，K8S负责容器编排和容器管理

### docker和虚拟机的区别？

往深的讲了，面试官说不用讲这么深，简单说一下就行了

### 从使用角度来说

- 结构不同
- 部署速度
- 可移植性
- 隔离性

### K8S是做什么的，有没有使用过

容器编排的，之前在公司使用过，不过是一件部署的

全程17分钟，很快哈哈，光速挂难道？