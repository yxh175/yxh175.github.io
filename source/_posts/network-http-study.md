---
title: 计算机网络：HTTP知识点遗忘点记录
date: 2023-09-20 16:35:47
categories: 八股文
tags: 
- 计算机网络
- HTTP
---

记录一下自己HTTP的相关薄弱点。强化一下学习
文章内容参考[小林coding](https://xiaolincoding.com/network/2_http/http_interview.html#https-%E6%98%AF%E5%A6%82%E4%BD%95%E5%BB%BA%E7%AB%8B%E8%BF%9E%E6%8E%A5%E7%9A%84-%E5%85%B6%E9%97%B4%E4%BA%A4%E4%BA%92%E4%BA%86%E4%BB%80%E4%B9%88)

## HTTPs 的应用数据如何保证完整性

这个问题，主要考查的是，HTTPs构建完对话后，如何去实现数据的传递，因为HTTPs传递消息不会像HTTP那样随意的明文传输，它还需要一个对称加密的过程。
这里涉及TLS的两个协议：

- TLS握手协议： 四次握手算法，负责协商加密算法和生成对称密钥的。
- TLS记录协议：负责保护应用程序数据并验证其完整性和来源，所以对HTTP数据加密是使用记录协议

TLS记录协议主要负责消息（HTTP数据）的压缩，加密及数据的认证。
![TLS处理信息的过程](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/HTTP/%E8%AE%B0%E5%BD%95%E5%8D%8F%E8%AE%AE.png)

过程如下：

- 消息分割：消息被分割成多个片段
- 片段压缩：消息通过压缩算法进行压缩
- MAC生成：通过Hash算法对压缩片段生成MAC并追加到尾部
- 对称加密：通过对称密钥进行加密
- 生成报文：由数据类型、版本号、压缩后的长度组成最终报文

## HTTP/2做了什么优化？

HTTP/2协议是基于HTTPS的，所以HTTP/2的安全性也是有保障的

![HTTP各版本架构](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/HTTP/25-HTTP2.png)

根据这些图可以看到HTTP/2相比于HTTP/1.1的一些改进

- 头部压缩：`HPACK`算法，通过维护一个头信息对应信息表，用索引号来表示字段。
- 二进制格式： 头信息和body都采用二进制传输
- 并发传输：引用Stream概念，多个Stream在一条TCP中复用。通过Stream ID 让接收方变得排序，服务端可以乱序发送
- 服务器推送：服务器会主动向客户端推送可能会用到的资源

## HTTP/2有什么缺陷？

HTTP/2 通过Stream的并发能力，虽然解决了HTTP/1队头阻塞的问题，但还是存在TCP的队头阻塞。因为HTTP/2是基于TCP的。TCP 是字节流协议，TCP 层必须保证收到的字节数据是完整且连续的，这样内核才会将缓冲区里的数据返回给 HTTP 应用，那么当「前 1 个字节数据」没有到达时，后收到的字节数据只能存放在内核缓冲区里，只有等到这 1 个字节数据到达时，HTTP/2 应用层才能从内核中拿到数据，这就是 HTTP/2 队头阻塞问题。

![stream丢失片段](https://cdn.xiaolincoding.com/gh/xiaolincoder/network/quic/http2%E9%98%BB%E5%A1%9E.jpeg)

当通过TCP发送多个包的时候，如果有一个包在网络中丢失，那么就会导致之后的内容会在内核中缓存。直到丢失的包收到。应用层才能在内核中取到结果
![内核运行图](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/http3/tcp%E9%98%9F%E5%A4%B4%E9%98%BB%E5%A1%9E.gif)

所以，一旦发生了丢包现象，就会触发 TCP 的重传机制，这样在一个 TCP 连接中的所有的 HTTP 请求都必须等待这个丢了的包被重传回来。

## HTTP/3 做了哪些优化？
