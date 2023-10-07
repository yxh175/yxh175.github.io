---
title: GO：并发编程 面试重点
date: 2023-09-21 15:36:09
categories:
- 八股文
tags:
- Golang
- 并发编程
---

## Mutex是什么？怎么go中使用？

Mutex 是go语言中的一种互斥锁，作用是给临界区的数据加锁，保证一次只能有一个协程去访问某个数据

在go语言中，通过`sync.Mutex`实现Mutex锁，Mutex内部是一个结构体
```go
type Mutex struct {
	state int32
	sema  uint32
}
```
通过Locker来保证互斥锁， 实战一下

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// mutex锁
	mutex := sync.Mutex{}
	var wg sync.WaitGroup
	res := 0
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			// 如果不加会是一个无法确定的输出
			mutex.Lock()
			res += 1
			mutex.Unlock()
		}()
	}
	wg.Wait()
	fmt.Println(res)
}

```
通过Mutex保证自增不会异常。

### 为什么不加Lock会异常？

这个涉及到了加法运算的问题了，加来说一个CPU在加的过程是在加法器上算的，如果此时切换到其他协程上去，这个值就还没有赋值上，其它的协程就进行了累加，但是是基于原本的结果。下次切换没加上的这个协程时就是赋值原先的结果。

## Mutex 锁有什么模式？

### 正常模式（非公平锁）

正常模式即是，等待队列中的协程按照FIFO的顺序等待，唤醒的goroutine不会直接获得锁，而是和当前新请求的goroutine竞争锁，但是新请求的goroutine是运行在CPU上的，所以刚唤醒的goroutine很大概率会竞争失败，加入到等待队列的前面

### 饥饿模式（公平锁）

为了解决等待goroutine队列的长尾问题，饥饿模式下，直接由unlock把锁交给等待队列中排在第一位的goroutine中，同时饥饿模式下goroutine不会参与抢锁也不会进入自旋状态，会直接进入等待队列的尾部，这样可以防止goroutine一直抢不到锁

### 何时进入饥饿模式？

- 当一个 goroutine 等待锁时间超过 1 毫秒时

### 何时退出？

当waiter队列获取到锁且满足以下任意条件

- 当前队列只剩下一个 goroutine 的时候，Mutex 切换到饥饿模式。
- 它等待获取锁的时间 < 1ms


### 总结

对于两种模式，正常模式下的性能是最好的，goroutine 可以连续多次获取锁，饥饿模式解决了取锁公平的问题，但是性能会下降，这其实是性能和公平的一个平衡模式。

## 自旋锁和等待锁

- 自旋锁就是依旧占用CPU，等自旋结束后就会直接进行工作
- 等待锁就是释放CPU，等待被唤醒时使用CPU，不占用额外的资源


## RWMutex 锁

- RWMutex 是单写多读锁，该锁可以加多个读锁或者一个写锁
- 写锁会阻止其他 Goroutine（无论读和写）进来，整个锁由该 Goroutine独占
- 适用于读多写少的场景
- RWMutex 的读锁或写锁在未锁定状态，解锁操作都会引发 panic

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	m := 1
	rw := sync.RWMutex{}
	var wg sync.WaitGroup
	wg.Add(3)

	go read(&m, &rw, &wg)
	go write(&m, &rw, &wg)
	go read(&m, &rw, &wg)

	wg.Wait()

}

func read(m *int, rw *sync.RWMutex, wg *sync.WaitGroup) {
	defer wg.Done()
	rw.RLock()
	fmt.Println(*m)
	time.Sleep(time.Second)
	rw.RUnlock()
}

func write(m *int, rw *sync.RWMutex, wg *sync.WaitGroup) {
	defer wg.Done()
	rw.Lock()
	*m = 2
	fmt.Println(*m)
	rw.Unlock()
}

```