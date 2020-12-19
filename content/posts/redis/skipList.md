---
title: "redis数据结构之跳跃表"
date: 2020-12-19T21:42:07+08:00
draft: false
tags: ["reids"]
keywords: ["redis", "设计与实现", "跳跃表", "skipList"]
---

跳跃表（skiplist）是一种有序数据结构，他通过节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

跳跃表支持平均O（logN），最坏O（N）复杂度的节点查找，还可以通过顺序操作来批量处理节点。**在大部分的情况下，跳跃表的效率可以和平衡树相媲美，并且因为跳跃表的实现比平衡树更加简单，所以redis使用跳跃表来替代平衡树**。

redis只在两个地方使用了跳跃表，一是实现有序集合键，二是在集群节点中用作内部数据结构。

### Redis跳跃表的实现

```
//跳跃表
typedef struct zskiplist {
	//表头节点和表尾节点
	struct zskiplistNode *head, *tail ;
	//表中节点的数量
	unsigned long length ;
	//表中层数最大的节点的层数
	int level ;
} zskiplist;

//跳跃表节点
type struct zskiplistNode {
	//层
	struct zskiplistlevel {
		//前进指针, 用于从表头向表尾方向访问		
		struct zskiplistNode *forward ;
		//跨度， 用于计算排位
		unsigned int span ;
	} level [ ] ;

	//后退指针，用于从表尾向表头方向访问
	struct zskiplistNode *backward;
	//分值
	double socre ;	
	//成员对象
	robj *obj ;
} zskiplistNode ;
```

### 跳跃表节点层高度的计算方式

每次创建一个新的跳跃表节点，程序都会根据幂次（越大的数出现的概率越小）定律随机生成一个介于1和32之间的值作为level数组的大小，这个大小就是层的“高度”。

```
// level计算伪代码
randomLevel() {
	level = 1 ;
	//random 返回一个[ 0......1 ) 的随机数，p = 1/4 ；maxLevel = 32
	while ( random() < p && level < maxLevel) {
		level++ ;
	}
	return level ;
}
```

### 分值与成员

分值是一个double类型的浮点数，跳跃表的所有节点都是按分支从小到大来排序。

成员对象是一个指针，他指向一个字符串对象，而字符串对象则保存着一个SDS值，各个节点的分值可以相同，但成员对象必须不同。在分值相同时，节点按照成员对象的大小来排序

