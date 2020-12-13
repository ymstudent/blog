---
title: "redis数据结构之链表"
date: 2020-12-13T13:01:53+08:00
draft: false
tags: ["redis"]
keywords: ["redis", "设计与实现", "链表"]
---

链表被广泛用于实现redis的各种功能，如：列表键，发布与订阅，慢查询，监视器等

### Redis链表的结构

```
//单节点
typedef struct listNode {
	//前置节点
	struct listNode * prev ;
	//后置节点
	struct listNode * next ;
	//节点值
	void * value ;
} listNode ;

//redis链表
typedef struct list {
	//表头节点
	listNode * head ;
	//表尾节点
	listNode * tail ;
	//链表所包含的节点数量
	unsigned long len ;
	//节点值复制函数
	void * (*dump) (void *ptr) ;
	//节点值释放函数
	void * (*free) (void *ptr) ;
	//节点值对比函数
	int (*match) (void *ptr, void *key) ;
} list ;
```

### Redis链表的特性

* 双端：带有前后节点的指针，获取某个节点的前置节点与后置节点的复杂度都是O(1)

* 无环：表头节点的prev指针与表尾节点的next指针都指向NULL，对链表的访问以NULL为终点

* 带表头指针与表尾指针：通过list结构的head指针与tail指针，程序获取链表表头节点和表尾节点的复杂度为O(1)

* 带链表长度计数器：程序使用list结构的len属性来记录持有的节点数量，程序获取链表中节点数量的复杂度为o(1)

* 多态：链表节点使用void* 指针来保存节点值，并通过list结构的dup,free,match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值

### 链表的优点与缺点

双端链表的优势在于便于在表的两端进行push和pop操作，在插入节点上复杂度很低，但它内存开销比较大。

首先，双端链表每个节点除了要保存数据之外，还要保存2个额外的指针；其次，双端链表的各个节点是单独的内存块，地址不连续，节点多了容易产生内存碎片