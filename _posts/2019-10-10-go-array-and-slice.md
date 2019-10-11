---
title: 深入理解Slice
key: 10010
tags: Go
layout: article
category: blog
comment: true
date: 2019-10-09 17:35:00 +08:00
modify_date: 2019-10-09 17:35:00 +08:00
---

## Go和C数组的区别
- 在Go中，数组名代表一组值，而C数组名是数组首地址
- 在Go中，数组当做参数是值拷贝，会把数组中所有的元素都拷贝过去
- 在GO中，数组长度也是类型的一部分，如[10]int和[20]int是不同的数据类型

## Slice原理

### struct

slice的结构大致如下：

```go
type slice struct {
  ptr *Elem //指向底层数组的指针
  len int //可容纳最大长度
  cap int //申请底层数组的长度
}
```

以语句 `s := make([]int, 5, 10)` 为例，包含如下信息：
- s指向了一个类型为[10]int的数组
- s当前容量为5，初始值为5个0，可扩容至10
- 如make不指明cap，则cap和len一致

### slicing
对一个slice进行切片，不会进行值拷贝，只是在底层数组上指针的移动。所以如果对切片后的slice进行修改操作，原slice也会相应的变化。
```go
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:] // e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

### 扩容len
```go
s := make([]int, 5, 200) //len(s) == 5
s = s[:cap(s)] //len(s) == 200
```
- slice不能扩容超过cap，既不能超过底层数组的长度
- 下标访问，index不能超过len，否则会runtime panic

### 扩容cap
如果cap不够，此时会发生内存拷贝和迁移，内置append函数中就包含了动态扩容，以slice s为例，大致过程如下：
```go
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
copy(t, s)
s = t
```
先申请一块两倍于当前s容量的数组，然后进行值拷贝，最后s指向新的数组

## vector原理

vector就比较贴近内存了, C++的capacity是申请的内存空间可最大容纳的元素数量，当超过capacity会进行内存拷贝迁移，size就是当前内存空间的元素数量

```c++
std::vector<int> vec;
vec.size(); //已用长度
vec.capacity(); //总共可容纳的元素数量
```

## 参考
- [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
- [Effective Go](https://golang.org/doc/effective_go.html#slices)