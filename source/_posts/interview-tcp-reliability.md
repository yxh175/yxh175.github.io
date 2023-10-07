---
title: 面试重点：TCP如何保证可靠性
date: 2023-10-03 17:32:21
categories:
- 八股
tags:
- TCP
---

文章内容主要参考：[小林coding](https://xiaolincoding.com/network/3_tcp/tcp_feature.html#%E8%B6%85%E6%97%B6%E9%87%8D%E4%BC%A0)

## TCP哪些机制保证了可靠性

- 重传机制：通过对未送达信息的重新传输，最终保证每个数据都能抵达到对端
- 滑动窗口：通过窗口增加传输的效率，可以等待一个范围的包
- 流量控制：发送方会根据对方的滑动窗口来控制自己的发送速度
- 拥塞控制：网络传输过程丢包严重也会通过降低发送速度防止出现恶性循环

## 重传机制

重传机制是为了保证，网络中报丢失后，能通过传输机制，最终保证报的正常传输

- 超时重传
- 快速重传
- SACK方法
- D-SACK方法

### 超时重传

超时重传的机制比较简单，只需要在每次发送时设置一个定时器，在定时器超时时没有收到ACK的话就会重发数据。这就是所谓的超时重传。

![超时重传](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/5.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

#### 重传时间RTO（Retransmission Timeout）设置多长合适

RTO较大会导致重发检测较慢，一次重发要好久，RTO较小会导致误判丢包次数过多，形成网络拥塞

最好设置刚好大于一个RTT往返时延

### 快速重传

所谓的快速重传，即是不依赖时间，而是根据丢包次数决定。

![快速重传](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/10.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

快速重传中，当一个包被接受时，接受方就会返回ACK + 下一个需求的序列号。这样如果一个包丢失时，就会返回返回一个最左边未收到的序列号。这样多次后，发送方就会知道有一个包丢失了，就会快速的重传

快速重传解决了超时重传时间的问题，但是也衍生了另外一个问题。即重传最左边的一个，还是重传所有问题。为此工程师又设计了`SACK`重传机制

### SACK重传

SACK( Selective Acknowledgment) 即选择性确认
这种方式是在TCP头部中增加了一个SACK的参数。它将已收到的数据的信息发送给发送方。这样发送方就会知道哪些数据丢失了，哪些数据传输成功了。

![SACK重传](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/11.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

SACK 会将最左侧的窗口保留，相当于告知发送方当前已经接受的数据。在ACK三次相同值触发快速重传时，传送方就能根据ACK的值和SACK的值推断出缺失的部分。

### D-SACK重传

SACK是为发送方丢包设计的保险协议。而丢包过程往往是双方的问题。当接受方的ACK包出现丢失或发送方出现延迟发送，就会出现重复数据的传输。为此TCP又提出了D-SACK重传模式。

小林coding中很好的用图总结了两个异常情况

![ACK包丢失](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/12.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

此时ACK多次丢失，导致发送方错误的任务自己的包没有发送成功，在RTO时间到的时候会触发超时重传。此时就会发送重复的数据。此时接受方回复了一个ACK大于SACK情况。这种情况就是D-SACK告知发送方数据重复了

![ACK包延迟](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/13.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

发送方延迟发送，导致发送方没有收到ACK的报文，所以它又重复了一次发送。这时候就出现了一次重复发送数据的情况。

这时也是一个D-SACK的情况，ACK的值大于SACK的范围。此时表示数据是重复的。此时发送方根据自己的发送记录，可以推测出，数据包在传输的时候出现了延迟。
D-SACK的好处有如下：

- 可以让发送方知道，发出去的包是丢了，还是ACK丢了
- 可以知道发送的包是不是网络延迟了
- 可以知道网络中是不是把包复制了

## 滑动窗口

### 什么是滑动窗口

算法中滑动窗口的表现是一个一定大小的窗口，在整个数据的遍历过程像一个窗口滑动一样。TCP引入滑动窗口是为了实现多个包的传输，而不会因为一个一个的顺序接收而导致阻塞。如果是一个发送对应一个ACK，通信效率就会受到往返时延的瓶颈

有了窗口就可以无需等待确认应答，而可以继续发送大量的数据，即使有个ACK丢失了，也可以通过ACK的最大值表示确认应答位置。可以这么理解ACK的值就是告诉当前发送方我的最左窗口位置

### 窗口大小由哪一方决定？

TCP中存在一个字段`Windows`，也就是窗口大小。
这个字段是接收方为了告诉发送端自己的缓冲区大小，发送端就可以根据这个Window的大小来控制自己发送的速度。而不会导致接收端处理不过来这些包

发送方的发送数据大小不能超过接受方的窗口大小，否则接收方就无法正常的接受数据

### 发送方的滑动窗口

发送方的滑动窗口由四个部分组成

- 确认收到的部分
- 已发送但未收到的部分
- 未发送但在接收范围的部分
- 未发生且超过接收方范围的部分

![发送方滑动窗口](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/19.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

这四个部分通过推理可以得知接收方的接收情况

这个四部分的分割需要三条边界进行分割，分别是：

- `SND.UNA` 绝对指针：接收方最后一个ACK的位置
- `SND.NXT` 绝对指针：发送方当前发送的位置
- `SND.UNA + SND.WND` 相对指针，通过接受方发送窗口大小的偏移量计算

还可以发送数据的窗口大小 = `SND.WND - (SND.NXT - SND.UNA)`

### 接收方的滑动窗口

接收方的滑动窗口分为三个部分

- 成功接收并确认的数据
- 未收到数据但可以接收的数据
- 未收到的数据并不可以接收的数据

![小林coding](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/20.jpg)

通过两个指针去控制

- RCV.WND 相对指针表示接收窗口大小
- RCV.NXT 是一个绝对指针，它指向发送方发送来的下一个数据字节的序列号

### 接收窗口和发送窗口的大小是相等的吗？

并不完全相等，但是是约等于的。为什么不能保证完全相等是因为TCP连接中两端存在时延

## 流量控制

发送方不能无脑的发数据给接收方，要考虑接收方处理能力
如果无脑的发送数据给对方，但对方处理不过来，就会导致重发机制的触发，对网络资源进行不必要的浪费

为此TCP提供了**流量控制**来保证发送方能根据接收方的实际接收能力控制发送的数据量。

通过控制窗口大小，来判断当前要发送多少数据

### 操作系统缓冲区与滑动窗口的关系

一般来说发送窗口和接收窗口所存放的字节数，都是放在操作系统内存缓冲区中，操作系统的内存缓冲区的大小会被操作系统调整，当进程没办法及时读取缓冲区的内容时，也会对缓冲区造成影响

#### 操作系统缓冲区如何影响窗口？

可以有两个例子来举。可以通过图解很清楚的看到这个过程，图解来自小林coding讲的很清楚

考虑以下场景：

- 客户端发送，服务端接收，初始窗口都是360
- 服务端繁忙。收到数据后应用层不能及时读取数据

![缓冲阻塞1](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/22.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

1. 客户端发送140字节的数据，可用变成220
2. 服务端收到140字节数据，但是进程只读取40字节，还有100占用缓冲区，此时要让窗口收缩至260，发送确认信息，并告知当前窗口大小
3. 客户端收到新的窗口大小后，就更新自己的窗口大小
4. 客户端发送180字节的数据
5. 服务端收到180字节的数据，因为进程阻塞就一点也没有读，此时窗口缩减至80，发送确认信息，并告知当前窗口大小
6. 客户端收到新的窗口大小，更新自己的窗口大小
7. 客户端发送80字节
8. 服务端收到80字节后依旧是没有进行读取，此时窗口缩减为0，发送确认信息，并告知当前窗口大小
9. 客户端接收到新的窗口，此时窗口为0，不进行发送

还有一种情况是因为操作系统资源不足导致数据丢包。

![数据丢包](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/23.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

因为内存不足，会到导致，客户端误认为自己的包发送成功，将自己的窗口更新到一个诡异的负数

### 窗口关闭

在流量控制中，接收方通过对发送方窗口大小的约束实现流量控制，当窗口大小为0时，会阻止发送方给接收方传递数据，直到窗口变为非0，这个就是窗口关闭

#### 窗口关闭的潜在风险

可以看到窗口关闭的原因一般是接收方无法处理发送方发送的数据，而进行的流量控制。假设接收在一定时间后有了足够的窗口去接收数据后，会发送一个窗口大小，来通知发送方更新窗口。以此来发送数据。
假如这个数据因为某些原因导致发送失败。可能会导致两者进入死锁

#### TCP如何解决窗口关闭时，潜在的死锁现象？

其实思路也很简单，通过定时器来实现，发送增加一个探测定时器，每一段时间都会询问对方窗口是否增大

### 糊涂窗口综合症

所谓糊涂窗口综合症是每次发送的数据都很少，对TCP每次运输的开销比较大。一般发送在流量控制导致的窗口变小后，因为应用层读取数据不连续，每次只能清理几个字节的小窗口让发送方发送。

所以糊涂窗口综合症可能发生的情况是

- 接收方可以通告一个小的窗口
- 而发送可以发送小数据

既然如此那么糊涂窗口综合症的解决思路也是从这两点下手：

- 接收方不通告小窗口
- 发送发不发送小数据

#### 怎么让接收方不通告小窗口？

接收方调整为窗口大小小于（MSS）缓存空间的1/2时，就发送0

#### 怎么让发送方避免小数据呢？

延时处理

- 要等窗口大小 >= MSS 且 数据大小 >= MSS
- 收到之前发送数据的ack回包

满足以上两个条件才能进行发送

## 拥塞控制

### 为什么有了流量控制之后还要拥塞控制？

这个在面试中经常问到，也是在刚学习时容易搞混的一个问题。其实也很好理解，所谓流量是指两个端口的表现，拥塞是之间传递的问题。所以流量控制解决两端流量大小问题，拥塞控制解决网络传输途中的拥塞问题

如果网络出现拥堵，不减少发送速率而继续进行大量的数据发送，会导致整个网络中的阻塞更加严重，形成恶性的循环，其实这个在计算机网络中的网络模块有类似的场景。
因此TCP提出了拥塞窗口的概念，来进行拥塞控制

### 什么是拥塞窗口？和发送窗口的关系？

拥塞窗口cwnd是发送方维护的一个状态变量，它会根据网络的拥塞程度动态的变化。

在有拥塞窗口的情况下swnd = min(cwnd, rwnd)

拥塞窗口cwnd的变化规则

- 网络没有拥塞，cwnd会逐渐增大
- 网络出现拥塞，cwnd就会减少

### 如何直到当前网络是否拥塞呢？

一般来说没有按照期望时间内收到ACK应答报文，即发送超时重传就认为网络出现拥塞

### 拥塞控制的一些算法

- 慢启动
- 拥塞避免
- 拥塞发送
- 快速恢复

### 慢启动

慢启动是在TCP连接刚建立完成时要执行的一个算法，它的核心作用是防止新建立的连接需要合适的初始化值去启动。否则容易出现网络阻塞。

慢启动的规则很简单：当发送方每收到一个ACK，拥塞窗口cwnd就会+1，表现情况会看到cwnd每一轮会乘指数增加

一个慢启动的过程举例如下：

- 初始值 cwnd = 1
- 发送一个数据，收到一个ack, cwnd += 1,此时cwnd = 2
- 此时可以发送两个数据，收到两个ack，cwnd += 2 此时cwnd = 4
- 以此类推可以看到cwnd每一轮会层指数性的增长

![慢启动](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/27.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

#### 什么时候慢启动停止

`ssthresh` （slow start threshold）状态变量，

- 当cwnd < ssthresh 时使用慢启动
- 当cwnd >= ssthresh 时使用拥塞避免

### 拥塞避免

当拥塞窗口cwnd超过阈值就会进入拥塞避免。什么是拥塞避免呢？顾名思义，拥塞避免就是为了防止慢启动增长过大导致网络最终承受不了，所以要使用拥塞避免算法

拥塞避免算法的规则是：没收到一个ACK，cwnd会增加1/cwnd 也就是说表现形式是线性增长

![拥塞避免](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/28.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

### 拥塞发生

我们可以看到上面两个现象都没有对增长进行限制，总体还会继续增长，这就难免会出现拥塞。当出现拥塞时，就会出现数据包重传。前面我们提到数据包重传有两种机制

- 超时重传
- 快速重传

这两种重传机制的拥塞发生算法也不同

#### 超时重传下

这时ssthresh和cwnd都会变小

- ssthresh设为cwnd/2
- cwnd 设置为初始值

![超时重传的拥塞发生](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8F%AF%E9%9D%A0%E7%89%B9%E6%80%A7/29.jpg?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

这种算法会导致网络传输的急刹车。反应过于强烈

#### 快速重传下

快速重传将这种网络包丢失严重性没有看的很重，但也会对cwnd和ssthresh两个值进行修改，同时会调用快速恢复算法

- cwnd = cwnd /2
- ssthresh = cwnd
- 快速恢复

### 快速恢复

快速重传使用快速恢复算法是因为快速恢复算法认为，你还能收到3个重复的ACK，说明网络不是态糟糕

快速恢复算法如下：

- 拥塞窗口 cwnd = ssthresh + 3 （ 3 的意思是确认有 3 个数据包被收到了）；
- 重传丢失的数据包；
- 如果再收到重复的 ACK，那么 cwnd 增加 1；
- 如果收到新数据的 ACK 后，把 cwnd 设置为第一步中的 ssthresh 的值，原因是该 ACK 确认了新的数据，说明从 duplicated ACK 时的数据都已收到，该恢复过程已经结束，可以回到恢复之前的状态了，也即再次进入拥塞避免状态；

![快速恢复](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/%E6%8B%A5%E5%A1%9E%E5%8F%91%E7%94%9F-%E5%BF%AB%E9%80%9F%E9%87%8D%E4%BC%A0.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)