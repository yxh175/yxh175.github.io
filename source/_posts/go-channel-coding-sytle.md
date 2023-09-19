---
title: GO：Channel专项面试题
date: 2023-09-18 20:55:26
tags: 
- Golang
- Channel
---
## 协程打印123

使用协程多次交替打印123

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	ch1, ch2, ch3 := make(chan struct{}, 0), make(chan struct{}, 0), make(chan struct{}, 0)
	wg.Add(3)
	go func() {
		defer wg.Done()
		for i := 0; i < 10; i++ {
			<-ch1
			fmt.Print("1")
			ch2 <- struct{}{}
		}
		<-ch1
	}()

	go func() {
		defer wg.Done()
		for i := 0; i < 10; i++ {
			<-ch2
			fmt.Print("2")
			ch3 <- struct{}{}
		}
	}()

	go func() {
		defer wg.Done()
		for i := 0; i < 10; i++ {
			<-ch3
			fmt.Print("3")
			ch1 <- struct{}{} 
		}
	}()
	ch1 <- struct{}{}
	wg.Wait()
}
```

通过`channel`实现异步交替， 通过`waitGroup`实现主协程的等待

## 如何优雅进行协程退出

通过`ctx`进行管道通信，通过`select`进行监听

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	fmt.Println("job start")
	go worker(ctx)
	time.Sleep(5 * time.Second)
	cancel()

	time.Sleep(2 * time.Second)
}

func worker(ctx context.Context) {
	defer fmt.Println("work down")

	for {
		select {
		case <-ctx.Done():
			return
		default:
			time.Sleep(1 * time.Second)
			fmt.Println("working")
		}
	}
}

```

