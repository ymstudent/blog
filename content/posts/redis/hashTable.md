---
title: "redis数据结构之字典"
date: 2020-12-14T22:28:21+08:00
draft: false
tags: ["redis"]
keywords: ["redis", "hashtbale", "设计与实现"]
---

redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希节点，而每个哈希节点就保存了字典中的一个键值对。

### Redis字典的结构

```
//redis字典
typedef struct dict {
	//类型特定函数，保存了一簇操作特定类型键值对的函数
	dictType *type ;
	//私有数据，保存了需要传给哪些类型特定函数的可选参数
	void *privdata ;
	//哈希表，一般只使用ht[0]，ht[1]只会在rehash时使用
	dictht ht[2] ;
	//rehash索引，当rehash不在进行时，值为-1
	int trehashidx ;
} dict ;

//哈希表
typedef struct dictht {
	//哈希表数组，里面存放了若干个哈希节点
	dictEntry ** table ;
	//哈希表数组的大小
	unsigned long size ;
	//哈希表大小掩码，用于计算索引值，总是等于size - 1
	unsigned long sizemask ;
	//该哈希表已有节点数量
	unsigned long used ;
} dictht ;

//哈希节点
typedef struct dictEntry {
	//键
	void *key ;
	//值
	union {
		void *val ;
		uint64_tu64 ;
		int64_ts64 ;
	} v ;
	//指向下个哈希表节点，形成链表
	struct dictEntry *next ;
} dictEntry ;
```

### 哈希值与索引值的计算方式

要将一个新的键值对添加到字典里面时，程序需要先根据键值对计算出哈希值和索引值，然后根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上。

```
//使用字典设置的hash函数，计算键key的哈希值

hash = dict->type->hashFunction( key )

//使用哈希表的sizemask属性和哈希值，计算出索引值。 ht[x]在rehash时是ht[1]，正常时候是ht[0]

index = hash & dict->ht[x].sizemask
```

字典被当做数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法计算键的哈希值

### Redis哈希表解决键冲突的方式

当有两个或以上数量的键被分配到哈希表数组的同一个索引上时，我们称这些键发生了冲突（collision）。

Redis 的哈希表使用链地址法来解决键冲突：每个哈希表节点都有一个 next 指针，被分配到同一个索引上的多个键值对可以通过这个 next 指针形成一个单项链表，这样就解决了键冲突问题。

因为哈希表节点没有指向链表表尾的指针，所以为了速度考虑，程序总是将新节点添加到链表的表头位置（复杂度为O(1)），排在其他已有节点前面。

### rehash

为了让哈希表的负载因子维持在一个合理的水平，程序需要对哈希表进行相应的扩展或收缩。

* 当满足以下任意一个条件时，程序会自动对哈希表进行扩展

  	* 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且哈希表的负载因子大于等于1
  	* 服务器在执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且哈希表的负载因子大于等于5
* 当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作
* 负载因子计算公式：load_factor = ht[0].used / ht[0].size //哈希表已保存节点数/哈希表大小

哈希表的扩展和收缩操作通过 rehash 来完成。rehash 简单来说就是将 ht[0] 哈希表中的键值对重新计算哈希值与索引值后放到到 ht[1] 指定的位置中去，但因为哈希表中保存的键值对数量可能达到数千万，如果一次性将这些键值对全部 rehash 到 ht[1]，服务器将会阻塞相当长一段时间，所以 redis 的 rehash 操作是分多次、渐进式的完成的。具体的 rehash 操作如下：

* 为 ht[1] 哈希表分配空间：
  	* 如果是扩展操作，那么 ht[1] 的大小为第一个大于等于 ht[0].used \* 2 的 2 ^ n。eg: ht[0] 的当前值为4， 4*2=8，而 8（2^3）恰好是第一个大于等于 4 的 2 的 n 次方，所以程序会将ht[1]哈希表的大小设置为 8
  	* 如果是收缩操作，那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2 ^ n
* 在字典中维持一个索引计数器变量 rehashidx，并将它的值设置为0，表示 rehash 工作正式开始
* 在 rehash 进行期间，每次对字典进行添加，删除，查找，更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx 索引上的所有键值对 rehash 到 ht[1]，当 rehash 工作完成后，程序将 rehashidx 属性的值增一
* 随着字典操作的不断进行，最终在某个时间点上，ht[0] 的所有键值对都会被 rehash 到 ht[1]，这时程序将 rehashidx 属性的值设置为 -1，表示 rehash 已完成
* 在 rehash 操作完成后，程序释放 ht[0]，将 ht[1] 设置为 ht[0]，并在 ht[1] 创建一个新的空白哈希表，为下一次 rehash 做准备

### 渐进式 rehash 执行期间的哈希表操作

在渐进式 rehash 执行期间，字典的删除，查找，更新等操作都会在两个哈希上进行，例如，要在字典里面查找一个键的话，程序首先会在 ht[0] 里面进行查找，如果没有找到的话，就会继续在 ht[1] 里面进行查找。

在渐进式 rehash 执行期间，新添加到字典的键值对都会被放在 ht[1] 哈希表中。