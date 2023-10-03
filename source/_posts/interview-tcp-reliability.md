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


