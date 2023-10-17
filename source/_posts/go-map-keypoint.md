---
title: GO底层剖析：Map底层实现
date: 2023-10-16 18:42:00
categories:
- 八股
tags:
- Golang
---

## Map

Map在面试中被问到的东西挺多的，常见的就有

- map并发安全
- map底层实现
- map扩容规则

map其实可以理解为一种哈希表的数据结构。

### 如何设计一个Map

Map既然是一种哈希表的数据结构，通常在设计它的时候需要注意两个点

- 哈希函数的选择
- 哈希冲突的解决

那么golang在这里是如何解决这些问题的呢？

### Golang中Map的设计

首先先翻看一下Map的源码实现

在Go的`runtime.hmap`是最核心的结构体

```go
type hmap struct {
	count     int   // 表示当前hash表的元素数量
	flags     uint8     
	B         uint8     // 表示当前buckets的数量，但是B是次方上的 即 len(buckets) == 2 ^ B
	noverflow uint16    
	hash0     uint32    // hash0 是哈希的种子，为hash函数带来随机性

	buckets    unsafe.Pointer       // 当前桶的指针
	oldbuckets unsafe.Pointer       // 扩容前保存的buckets字段
	nevacuate  uintptr

	extra *mapextra
}

type mapextra struct {
	overflow    *[]*bmap
	oldoverflow *[]*bmap
	nextOverflow *bmap
}
```

所以一个hmap的结构如下

![hmap结构](https://s2.loli.net/2023/10/16/uxlRmjYQFt8JToc.png)

主要注意的是绿色部分，它表示桶的数组指针部分。bucket的结构又是什么样的呢？

bucket的主要由`bmap`结构体实现

```go
type bmap struct {
	tophash [bucketCnt]uint8
}

// bmap的相关方法
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > emptyOne && h < minTopHash
}

func (b *bmap) overflow(t *maptype) *bmap {
	return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-goarch.PtrSize))
}

func (b *bmap) setoverflow(t *maptype, ovf *bmap) {
	*(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-goarch.PtrSize)) = ovf
}

func (b *bmap) keys() unsafe.Pointer {
	return add(unsafe.Pointer(b), dataOffset)
}
```

这里unsafe.Pointer可以理解为一个指针数组，add就类似于append，类似一个操作指针的过程，只不过都是封装好的

这段可以抽象成这样一个图

![bucket结构](https://s2.loli.net/2023/10/16/p6KieT3J9Wjkmhy.png)

这里key和value分开放是因为key和value的类型往往是不同的，所以分开放防止padding导致内存增大

那知道这两个结构了，它内部真实的使用的整体关系是什么呢？

![map整体结构](https://s2.loli.net/2023/10/16/nukTtmqfAhDzpIW.png)

低八位查找桶，高八位查找key

### Map扩容实现

在`runtime.map.go`中详细的写了各种情况的扩容情况

```go
// Picking loadFactor: too large and we have lots of overflow
// buckets, too small and we waste a lot of space. I wrote
// a simple program to check some stats for different loads:
// (64-bit, 8 byte keys and elems)
//  loadFactor    %overflow  bytes/entry     hitprobe    missprobe
//        4.00         2.13        20.77         3.00         4.00
//        4.50         4.05        17.30         3.25         4.50
//        5.00         6.85        14.77         3.50         5.00
//        5.50        10.55        12.94         3.75         5.50
//        6.00        15.27        11.67         4.00         6.00
//        6.50        20.90        10.79         4.25         6.50
//        7.00        27.14        10.15         4.50         7.00
//        7.50        34.03         9.73         4.75         7.50
//        8.00        41.10         9.40         5.00         8.00
//
// %overflow   = percentage of buckets which have an overflow bucket
// bytes/entry = overhead bytes used per key/elem pair
// hitprobe    = # of entries to check when looking up a present key
// missprobe   = # of entries to check when looking up an absent key
//
// Keep in mind this data is for maximally loaded tables, i.e. just
// before the table grows. Typical tables will be somewhat less loaded.
```

判断扩充的条件，就是哈希表中的加载因子，加载因子是一个阈值，一般表示为：散列包含的元素数 除以 位置总数。是一种“产生冲突机会”和“空间使用”的平衡与折中：加载因子越小，说明空间空置率高，空间使用率小，但是加载因子越大，说明空间利用率上去了，但是“产生冲突机会”高了

每个哈希表都会有有一个加载因子，数值超过了加载因子就会进行扩容
golang的计算公式为`count/2^B > 6.5` 当key和B的关系出现这种情况时或者溢出桶太多也会导致扩容，就代表需要扩容

根据触发的机制不同，也会触发两种扩容

#### 等量扩容

这种是因为大量删除导致，溢出桶过多，每个桶的实际使用没有达到，这个时候就需要通过等量扩容/缩容去重新整理一下数据，使溢出桶中的数据重新紧凑的放在普通的bucket桶中，避免不必要的空间浪费

#### 渐进式扩容

这种是因为超过了加载因子，创建了一个新的桶指针数组，但是在迁移过程中如何一次性迁移，一是会造成很大CPU占用，而是需要进行大量的垃圾清除。因此Go采用渐进式的数据迁移，每次最多迁移两个bucket的数据到新的buckets中（一个是当前访问key所在的bucket，然后再多迁移一个bucket）。