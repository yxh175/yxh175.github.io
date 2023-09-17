---
title: Mysql 日志系统
date: 2023-09-16 16:03:52
categories:
- 八股文
tags: 
- mysql
- 日志
---

# Mysql日志系统

mysql的三大日志

- undo log（回滚日志）： 是innodb存储引擎层生成的日志，实现了事务的原子性，主要用于事务回滚和MVCC
- redo log（重做日志）: 将命令依次记录，用来实现事务的持久性。主要用于掉电等故障恢复
- bin log（归档日志）：是Server层生成的日志，主要用于数据备份和主从复制

## undo log

### undo log 是什么？ 为什么要有undo log？

`undo log` 就是反着的`sql`命令。你删除了我增。你加我删。保证一会能回滚。
因为在`mysql`中执行增删改的命令时，是隐式的开启事务的。事务如果开启出现了异常也能够正常的回滚。保证了`mysql`的 `A` 原子性

MVCC的实现。mysql多版本并发控制的必须需要`undo log` 和 `ReadView`这两个。

一个`undo log`的格式日志如下

- 通过`trx_id`可以知道该记录被哪个事务修改
- 通过`roll_pointer`指针来将这些undo log串联在一起

![undo log版本链](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%89%88%E6%9C%AC%E9%93%BE.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

通过这个版本链可以很好完成MVCC的多版本。同时mysql的两个隔离级别可重复读和读已提交则是由ReadView的方式决定的

- 读已提交： 隔离级别第二。实现方法是每次Select时都是生成一个新的ReadView。别的线程在更新数据后，下次的ReadView都不会相同
- 可重复读：隔离级别第三。与读已提交不同的是，是以事务进行划分的，在一个事务里，第一次Select生成的ReadView，在接下来不操作数据的情况下会进行继续使用

undo log的两大作用

1. 实现事务回滚，保障事务的原子性
2. 结合ReadView实现MVCC，实现高并发

### undo log 如何持久化

undo log 的数据在buffer pool里会记录到redo log 中，在通过redo log 刷盘实现持久化

### undo log 怎么记录的

开启事务后，如果使用了更新命令，就会把旧值记录下来。生成一个undo log, 然后写入buffer pool

## buffer pool

### buffer pool 是什么？为什么需要它？

buffer pool 是一个缓冲池， 通过建立在Server 和 磁盘之间的一个缓冲池。确保高频不需要重复的查询磁盘。

![缓冲池](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/%E7%BC%93%E5%86%B2%E6%B1%A0.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

Buffer pool的好处:

- 读取数据可以直接从缓冲池里读，加快时间
- 如果数据修改后，不会很快的进行磁盘I/O会有一个合适的时机刷进磁盘

### buffer pool 的缓冲结构？

InnoDB中数据的存储是使用16K的页进行存储的。为了方便对应。buffer pool中的数据也是以相同大小的页进行对应

在mysql刚启动时，会为buffer pool 分配多个16k大小的页。这些页作为即为缓存页

刚开始这些缓存页都是空闲的，只有真正要用的时候，就会出现缺页补充了

缓存页不仅仅会作为主要的数据页和索引页还有其他的一些页， 如：插入缓存页、undo页、自适应哈希索引、锁信息

![buffer pool页表示](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/bufferpool%E5%86%85%E5%AE%B9.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

### 查询一条记录， 就只需要缓冲一条记录吗？

肯定不是，刚才我们也说了，缓冲页对应着磁盘页的大小，所以每次查询一条记录，会把整个页都拿过来缓冲，再通过目录去查询记录

## redo log

### 为什么需要redo log?

刚才提到了在`Server`层和磁盘之间加上一个`buffer pool`，写进去的数据会先缓冲一下。看着似乎没什么问题，但是如果出现断电或者一些其他情况导致服务器宕机。那这个`buffer pool`缓冲的数据不都全寄了吗。

所以为了保证数据不会丢失，当有一条记录需要更新的时候，`InnoDB`会把`buffer pool`的数据页进行修改，并标记为脏页。然后用`redo log` 记录一下命令。更新才算完成。之后`InnoDB` 会用后台线程将`Buffer pool`的数据写到磁盘中 即`WAL`技术

`WAL`技术即Write-Ahead-Logging。兵马未动粮草先行。先把日志写好，之后在后台将缓冲数据写入。这样即使是断点了，redo log 也已经保存在物理盘里了。下次启动redo 就行了

![WAL技术流程](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/wal.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

### redo log 是什么？

redo log 就是物理日志，它直接记录了怎么操作表空间的哪些记录。在事务提交时，只要先将redo log 持久化，就不需要去等buffer pool的内容持久化了

如果系统不幸断电，redo log 也已经持久化，在重启mysql后就会进行恢复

### 被修改 Undo 页面，需要记录对应 redo log 吗？

需要的，任何从Server进入buffer pool中数据，都要生成redo log

### redo log 和 undo log 区别在哪？

redo log 是记录了完成后的记录状态。是为了持久化
undo log 是记录了更新前的记录状态。是为了原子性

### redo log 要写到磁盘，数据也要写磁盘，为什么要多此一举？

redo log 是追加写，所以磁盘操作时顺序写，而直接写到磁盘，需要查各种偏移地址来让找写的位置。磁盘操作就是随机读写。这样的读写肯定效率低。会拖累mysql整体的性能，写入数据慢，而且极大的增加还出现断电数据丢失的可能性

### redo log 是直接写入数据的吗？

并不是，redo log 本身也会使用一个buffer，每产生一个数据都会先加入buffer中，最后在写入到磁盘中

![redo log buffer](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/redologbuf.webp)

### redo log 什么时候刷盘？

redo log什么时候写入到磁盘中

- `Mysql`正常关闭
- `redo log buffer` 记录大于`buffer`内存关键的一半时
- 每隔一秒刷新一次
- 事务提交会将`redo log` 存入（由`innodb_flush_log_at_trx_commit`控制）

#### innodb_flush_log_at_trx_commit 怎么进行控制的

三种模式

- 0 事务提交后不刷入
- 1 事务提交后直接存入并刷盘
- 2 事务提交后交给写入redo log 文件，等待系统自动提交

![redo log 事务提交刷盘策略](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/innodb_flush_log_at_trx_commit.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

#### 这三个参数的应用场景是什么？

- 数据安全性：参数 1 > 参数 2 > 参数 0
- 写入性能：参数 0 > 参数 2> 参数 1

所以，数据安全性和写入性能是熊掌不可得兼的，要不追求数据安全性，牺牲性能；要不追求性能，牺牲数据安全性。


### redoLog 会写满吗？

高并发的情况下是会的。redo log 默认情况是由两个文件循环写实现。如果buffer太多没有刷盘，会让redo log 写满整个环，这时候mysql 就会阻塞来把脏页数据给持久化。这时就可以清理redo log里的不需要的数据了

## bin log

### 什么是bin log?

undo log 和 redo log 都是innoDB引擎生成的。undo 保证原子性，redo 保证持久化

bin log是做什么的呢。首先bin log 是在Server层中生成的。bin log会记录所有数据表结构变更及数据修改的情况, 但不会记录查询日志。

### 为什么有了bin log 还要redo log

bin log 是mysql 以前版本留下来的，用来存储命令的。用来进行归档的日志存储系统。没有cache-safe的能力。所以在innoDB提出的时候就增加redo log

### redo log 和bin log 的区别

1. 实现层不同

    - bin log是在server层实现，所有Mysql版本都能使用
    - redo log 仅在innoDB中使用
2. 记录内容不同

    - bin log 记录的是命令
    - redo log 记录的直接修改某个位置的日志

3. 读写方式不同

    - bin log 是直接读写在文件中， 文件满了就换个文件读
    - redo log 是循环读，需要不断刷新旧的日志

4. 用途不同

    - bin log 用于备份恢复，和主从复制， 保护整体数据
    - redo log 用于掉电故障的恢复， 保护cache级数据

### 如果不小心整个数据库的数据被删除了，能使用 redo log 文件恢复数据吗？

不是redo log，应该是bin log

redo log可能会擦除旧日志（因为它要保护的主要目标是buffer pool），bin log保存全量日志，且不会主动擦除

### 主从复制如何实现？

主从复制依赖于bin log。这个过程是异步的。主服务器不会管从服务器有没有写上。它只管用`log dump`线程发送就行

![主从服务](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E8%BF%87%E7%A8%8B.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

简单说一下就是：

- 写入binlog： 主服务在收到提交事务的请求后，会先写日志，在提交事务。更新存储引擎的数据，事务提交完成后，返回客户端操作成功
- 同步binlog: 从库从I/O线程连接log dump中的数据，接受binlog 日志，写入到自己的relay log日志中，并给主库返回一个响应
- 回放binlog： 从库的SQL线程会进行一个回放，将relay log中的操作，重新执行一遍实现主从数据的一致性

一般来说主从服务的部署是：**主服务器写，从服务器读**
![主从复制](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%BB%E4%BB%8E%E6%9E%B6%E6%9E%84.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

#### 从库越多越好吗？

不是， 从库越多，主库需要分担更多log dump线程去发送，资源消耗高

#### MySql 主从复制还有哪些模型?

- 同步复制： 主服务器等待从服务器同步完成。性能较差，可能会影响业务的使用
- 异步复制：默认情况，主库不管从库的数据拷贝情况，可能出现主库宕机，丢失数据的风险
- 半同步复制：只要有一部分同步成功返回响应了，就可以保证同步成功了，结合了两者的优点

### binlog 什么时候刷盘？

因为mysql 有事务的存在，一个事务的bin log是不能拆开的，所以mysql给每个线程都分配一个bin log cache 来记录事务的日志。这样每个事务就会隔离出来

![bin log 刷盘](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/binlogcache.drawio.png)

这里write 是缓存，fsync 是存入磁盘。

既然又有write又有fsync，同样的也存在性能和风险共存的情况

mysql提供sync_binlog参数进行选择

- 0： 只write， 由fsync决定什么时候写入
- n > 0： 每次提交n个事务，进行fsync

mysql 默认是0， 性能最高，风险最大

## 参考

[小林coding](https://xiaolincoding.com/mysql/log)