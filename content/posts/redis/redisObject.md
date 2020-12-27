---
title: "redis五种对象的底层实现"
date: 2020-12-27T17:57:11+08:00
draft: false
tags: ["redis"]
keywords: ["redis", "设计与实现", "redisObject", "五种对象"]
---

redis 使用对象来表示数据库中的键和值，键总是一个字符串对象，而值则是字符串对象、列表对象、哈希对象、集合对象、有序集合对象中的一个。下面是 redis 对象的数据结构：

```
//redis对象的结构
typedef struct redisObject {
	//类型
	unsigned type:4;
	//编码
	unsigned encoding:4;
	//指向底层实现数据结构的指针
	void *ptr;
	//引用计数
	int refcount ;
	//空转时长
	unsigned lru ;
} robj;
```

### 字符串对象

字符串对象根据保存值的不同,底层有三种实现方式:

* 整数值 ( int ) ： 值为整数且可以用 long 类型来表示，那么字符串对象就会选择 int 作为底层结构

* embstr 编码的简单动态字符串：值为字符串且字符串的长度小于等于 32 字节,字符串对象则选用 embstr 编码的简单动态字符串作为自己的底层实现

* raw 编码的简单动态字符串：值为字符串且字符串的长度大于 32 字节,字符串对象将会选择 raw 编码的简单动态字符串作为自己的的底层实现
* embstr 编码与 raw 编码的简单动态字符串的区别：embstr 的 redis object 和 SDS 在一块内存中，而 raw 则会为 redis object和 SDS 各申请一块内存。所以 embstr 分配内存和释放内存只需要调用一次对于函数就行，而 raw 则需要调用两次。

* ps: long double 类型的浮点数将被转化为字符串保存。

### 列表对象

* 压缩列表 ( ziplist )：保存的所有字符串元素的长度都小于 64 字节且元素的个数小于 512 个

* 双端链表 ( linkedlist )：上面任意一个条件不满足时，redis 就会使用双端链表作为哈希对象的底层实现

* ps： 上面两个条件值可以通过配置文件中的 list-max-ziplist-value 和 list-max-ziplist-entries 来修改

* **由于双端链表占用空间更多，所以 redis 在创建新列表时，会首先考虑压缩列表，并在有需要的时候，才从压缩列表转换到双端链表**

* 在 3.2 版本之后，redis 创建了一个新的数据结构 quickList 作为 list 的底层实现，quickList 是一个以 ziplist 为节点的双端链表，每个ziplist 包含的数据数量可以通过 list-max-ziplist-size 来设置

### 哈希对象

* 压缩列表: 键和值的长度都小于 64 字节且键值对的数量小于 512 个

* 字典 ( hashtable )： 上面任意一个条件不满足时，redis 将使用 hashtable 作为哈希对象的底层实现

* ps：上面两个条件可以通过配置文件中的 hash-max-ziplist 和 hash-max-ziplist-entries 来修改

### 集合对象

* 整数集合 ( intset )： 保存元素都为整数且元素数量不超过 512 个

* 字典：上面任意一个条件不满足时，redis 将使用 hashtable 作为集合对象的底层实现

* ps：上面元素数量可以通过配置文件中的 set-max-inset-entries 来修改

### 有序集合对象

* 压缩列表：保存元素数量小于 128 个且所有元素成员的长度都小于 64 字节

* zset结构 ( 跳跃表+字典实现，但使用OBJECT ENCODING查看编码时返回skiplist )：上述任意条件不满足时，redis 将使用 zset 结构来作为底层实现

* ps：上面两个条件可通过配置文件中的 zset-max-ziplist-entries 和 zset-max-ziplist-value 来修改