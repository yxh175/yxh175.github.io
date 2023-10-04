---
title: GO：反射实战编程
date: 2023-10-04 09:34:45
categories: 
- 编程实战
tags:
- reflect
---

`reflect`包在`Golang`语言中有重要的作用。反射在计算机科学领域中，是指一个应用能够进行自描述、自控制。Golang中实现了反射，反射机制就是在运行时动态的调用对象的属性和方法。

reflect中有两个重要的函数和类型

两个函数

- reflect.TypeOf  获取类型信息
- reflect.ValueOf  获取数据的运行时表示

两个类型

- reflect.Type
- reflect.Value

reflect.Type是反射包定义的一个接口，其中定义了一些不错的方法。这里列举出来一些，如果感兴趣可以翻一下源码。

```go
type Type interface {
	Align() int
	FieldAlign() int
	Method(int) Method
    // 获取当前类型对应方法的引用
	MethodByName(string) (Method, bool)
    // 获取当前类型方法的数量
	NumMethod() int

	...
    // 判断当前类型是否实现某个接口
	Implements(u Type) bool
    ...
}
```

`reflect.Value`类型则不同，它是一个结构体。而且也只有三个字段。它以结构体的方法实现的。

```go
type Value struct {
	typ *rtype
	ptr unsafe.Pointer
	flag
}


```

## interface 与reflect

在了解reflect需要先知道interface的作用。可以把接口理解成电脑的一个接口。只要你符合接口方法的要求，你就能接入这个接口。在go1.18版本前，很多开发者利用interface的特性实现泛型的效果。这是因为interface作为一个边界类型，可以实现类型的接受。反射作为一个自描述、自控制的代码块。所以作为边界的interface{}是必不可少的。interface的灵活性是一把双刃剑，反射作为一种元编程方式，可以减少重复代码。但是过度的使用会导致程序逻辑不清晰，且会运行缓慢。

## GO 反射三大法则

- interface{} 变量可以转换成反射对象
- 从反射对象可以获取interface{}变量
- 要修改反射对象，其值必须可设置

### 第一法则

反射是如何反射出对象的值并进行修改的。实际上，reflect过程中是发生了值传递，通过reflect.Value和reflect.Type进行接收

可以看以下ValueOf的源码, 我的go用的是1.20版本，这时使用的是any，如果你是1.18版本之前的话，这里就是interface{}。当然interface{}和any是相同的`type any = interface{}`

```go
func ValueOf(i any) Value {
	if i == nil {
		return Value{}
	}

	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i)

	return unpackEface(i)
}
```

可以看到这里进行了一个隐式的接口转换。

```go
func main() {
	test := "this is test"
	x := reflect.ValueOf(test)
	fmt.Println(reflect.TypeOf(x))
}
```

![第一法则](https://img1.imgtp.com/2023/10/04/xTXtYXTB.png)

这个x实际就是一个新的`reflect.Value`对象。`reflect.TypeOf` 和 `reflect.ValueOf` 相当于是一个桥梁连接了interface{}和反射对象

### 第二法则

第二法则即：从反射对象获取interface{}变量

这个比较容易理解，通过Interface转换成interface

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	test := "this is test"
	x := reflect.ValueOf(test)
	newX := x.Interface().(string)
	fmt.Println(newX)
	fmt.Println(reflect.TypeOf(newX))
}
```

```test
this is test
string
```

这段代码的输出结果，符合预期，反射对象被成功转换成变量

![第二法则](https://img1.imgtp.com/2023/10/04/VQDaMPcs.png)

通过这个方法就可以将反射转换成对象

### 第三法则

更新值的时候，要判断值是否可以更新

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	i := 1
	v := reflect.ValueOf(i)
	v.SetInt(10)
	fmt.Println(i)
}
```
上述代码初看貌似没有问题。但仔细看v时反射对象，对反射对象修改值，属于无效操作。程序为了防止错误就会崩溃

```test
panic: reflect: reflect.Value.SetInt using unaddressable value

goroutine 1 [running]:
reflect.flag.mustBeAssignableSlow(0x0?)
	C:/Program Files/Go/src/reflect/value.go:260 +0x85
reflect.flag.mustBeAssignable(...)
	C:/Program Files/Go/src/reflect/value.go:247
reflect.Value.SetInt({0xd966c0?, 0xe37968?, 0xd056a9?}, 0xa)
	C:/Program Files/Go/src/reflect/value.go:2233 +0x48
main.main()
	f:/goProject/go-learn/reflect/changeValueDemo/main.go:11 +0xb1
exit status 2
```

正确的修改原变量应该使用如下方法:

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	i := 1
	v := reflect.ValueOf(&i)
	v.Elem().SetInt(10)
	fmt.Println(i)
}
```

此时就能正确输出结果，所以有个要点，反射要传指针，反射对象要指向指针的值才可以修改值


