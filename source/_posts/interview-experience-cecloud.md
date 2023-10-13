---
title: 面试经验：中国电子云后端开发
date: 2023-10-13 17:15:06
cover: https://s2.loli.net/2023/10/13/QnksFYuVKzCDlPh.png
top_img: https://s2.loli.net/2023/10/13/MD2x49mQE76wJgu.png
categories: 
- 面经
tags:
- 面经
---

记录一下2023年10月13日中国电子云一面流程，整体流程很快，自我介绍，挖实习挖项目，问了一点点八股，然后就是反问环节，正式面试的时候整个流程17min，有点KPI的意思，反问的时候和面试官聊了很多，面试官人挺好的讲的很清楚。

## 从个人性格、学校经历、实习经历三个点进行自我介绍

我听得完直接按照自己之前背的个人介绍说了出来

## 挖实习

### 从什么角度去考虑重构的

单体服务转到微服务上去

### OCR实现流程的

简单的API调用，然后就说了其他的点

### 还有什么商业模型？

无，感觉面试官对实习做的东西不是很满意

## 挖项目

### 双token 怎么实现的，为什么要双Token？

双Token是AccessToken和RefreshToken两个token，通过两个token分别记录

### gin框架怎么使用的获取这两个token

通过中间件拦截，在中间件里的c.getHeader获取的

### 怎么保证安全性的？

安全性并不能绝对的保证，但是可以让他更安全。需要获取两个AccessToken和RefreshToken。AccessToken的过期时间短，RefreshToken的过期时间更长。
假设有人偷取了我的两个Token，但是RefreshToken的自动刷新也会

### 和单Token续约不行吗？

单Token续约存在一些问题，因为实际上的业务存在两种Token续约的情况，一个是用户不断的点击去续约Token，但是还有一种情况比如说用户一天内没有登录，或者一段时间没有操作。那这时候单Token就会有一些尬尴的处境

- 过期时间设置过长，会导致token无法保证安全性
- 过期时间设置过短，会导致token过期，导致用户长时间不操作需要频繁的去进行登录

## 八股

### channel底层怎么实现的？

如下是channel的底层代码，可以看到它是一个`hchan`结构体

```go
type hchan struct {
    qcount   uint           // 队列中的数据个数
    dataqsiz uint           // 环形队列的大小，channel本身是一个环形队列
    buf      unsafe.Pointer // 存放实际数据的指针，用unsafe.Pointer存放地址，为了避免gc
    elemsize uint16 
    closed   uint32 // 标识channel是否关闭
    elemtype *_type // 数据 元素类型
    sendx    uint   // send的 index
    recvx    uint   // recv 的 index
    recvq    waitq  // 阻塞在 recv 的队列
    sendq    waitq  // 阻塞在 send 的队列

    lock mutex  // 锁 
}
```

简单的用一个图去解释这`hchan`的主要参数的关系

![channel底层](https://s2.loli.net/2023/10/13/fqWkzD4RJXP8CYa.png)

主要的是一个环形队列，去保存传输的缓存数据

`sendx`和`recvx`表示这两个环形队列的位置
`recvq`和`sendq`表示两个等待队列的当前指针

### 两个协程之间如何通信？

平时写的时候是使用channel进行通信。通过make创建对应类型的channel进行通信

### GMP调度怎么实现的

基本的

- G Goroutine 的缩写，每次 go func() 都代表一个 G，无限制，但受内存影响。使用 struct runtime.g，包含了当前 goroutine 的状态、堆栈、上下文
- M 工作线程(OS thread)也被称为 Machine，使用 struct runtime.m，所有 M 是有线程栈的。M 的默认数量限制是 10000（来源），可以通过debug.SetMaxThreads修改。
- P Processor，是一个抽象的概念，并不是真正的物理 CPU，P 表示执行 Go 代码所需的资源，可以通过 GOMAXPROCS 进行修改。当 M 执行 Go 代码时，会先关联 P。当 M 空闲或者处在系统调用时，就需要 P

这个博客里讲的很好，之前也是看着这个理解GMP的调度。可以参考一下[GMP调度模式](https://blog.csdn.net/xmcy001122/article/details/119392934)

