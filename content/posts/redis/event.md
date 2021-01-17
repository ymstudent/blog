---
title: "Redis事件"
date: 2021-01-17T22:34:10+08:00
draft: false
tags: ["redis"]
keysword: ["redis", "events"]
---

Redis 服务器是一个事件驱动程序，服务器需要处理以下两类事件：

* 文件事件

* 时间事件

### 文件事件

Redis 基于 Reactor 模式开发的网络事件处理器被称为文件事件处理器（file event handler）。

#### 文件事件处理器的构成

文件事件处理器由四个部分组成：套接字，I/O多路复用程序，文件事件分派器（dispatcher），事件处理器。如下图：

<img src="https://ymfeb.cn/images/symrmF.png" alt="fe21b9a959ba877059361c622f2e39d8.jpeg" style="zoom:70%;" />

文件事件是对套接字操作的抽象，每当一个套接字准备好连接应答（accept）、读取（read）、写入(write)、关闭（close）等操作时，就会产生一个文件事件。

I/O 多路复用程序负责监听多个套接字，并向文件事件分发器传送那些生产了事件的套接字。因为同时监听了多个套接字，所以文件事件可能并发地出现，I/O 多路复用程序通过队列，以有序、同步、每次一个套接字的方式向文件事件分发器传送套接字。

文件事件分发器接收到 I/O 多路复用程序传来的套接字，根据套接字产生的事件类型，调用相应的事件处理器。

事件处理器是一个个函数，它们定义了某个事件发生时，服务器应该执行的动作。

#### I/O多路复用程序的实现

Redis 的 I/O 多路复用程序的所有功能都是通过包装常见的 select、epoll、evport、kqueue 这些多路复用函数库来实现的。Redis 为这些函数库实现了相同的 API，所以 I/O 多路复用程序的底层是可以互换的。

Redis 会优先选择时间复杂度为 𝑂(1) 的 I/O 多路复用函数作为底层实现，包括 Solaries 10 中的 evport、Linux 中的 epoll 和 macOS/FreeBSD 中的 kqueue，上述的这些函数都使用了内核内部的结构，并且能够服务几十万的文件描述符。

但是如果当前编译环境没有上述函数，就会选择 select 作为备选方案，由于其在使用时会扫描全部监听的描述符，所以其时间复杂度较差 𝑂(𝑛)，并且只能同时服务 1024 个文件描述符，所以一般并不会以 select 作为第一方案使用。

#### I/O多路复用程序中几个比较重要的API

```
//创建一个多路复用程序，并设置其能监听的套接字数量
aeEventLoop *aeCreateEventLoop(int setsize)
//将给定套接字的给定事件加入到I/O多路复用程序的监听范围之内，并对事件和事件处理器进行关联
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData)
//让I/O多路复用程序取消对给定套接字的给定事件的监听，并取消事件和事件处理器之间的关联
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)
//在给定事件内阻塞并等待套接字产生给定类型的事件，当事件成功或等待超时后，函数返回
int aeWait(int fd, int mask, long long milliseconds)
//在指定时间内，阻塞并等待所有被aeCreateFileEvent函数设置为监听状态的套接字产生事件，当至少有一个事件产生，或超时后，函数返回
int aeApiPoll(eventLoop, tvp);
//文件事件分配器，先调用aeApiPoll等待事件产生，然后遍历所有产生的事件，调用相应处理器来处理这些事件
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
```

#### 套接字事件的类型

I/O多路复用程序可以同时监听多个套接字的 ae.h/AE_READABLE 事件和 ae.h/AE_WRITEABLE 事件，这两种事件与套接字的关系如下：

* 当套接字变得可读时（客户端对套接字执行 write 操作或 close 操作），或者有新的可应答套接字出现时（客户端对服务器监听的套接字执行 connect 操作），套接字产生 AE_READABLE 事件。

* 当套接字变得可写时（客户端对套接字执行 read 操作），套接字产生 AE_WRITEABLE 事件。

多路复用程序允许服务器同时监听套接字的两种事件，当一个套接字可读又可写时，服务器将会先读套接字，后写套接字。

#### 文件事件的处理器

Redis 为文件事件编写了多个处理器，分别用于实现不同的网络通信请求。在这些处理器中最常用的要数**连接应答处理器、命令请求处理器和命令回复处理器**。

* 连接应答处理器用于对连接服务器的各个客户端进行应答。在 Redis 服务器初始化时，程序会将连接应答处理器与服务器监听套接字的 AE_READABLE 事件关联起来，当客户端使用 connnect 函数连接服务器监听的套接字时，套接字就会产生 AE_READABLE 事件，引发连接应答处理器执行相应处理。

* 命令请求处理器负责从套接字中读入客户端发送的命令请求的内容。在客户端成功连接到服务器后，服务器会将客户端套接字的 AE_READABLE 事件与命令请求处理器关联起来，当客户端向服务器发送命令请求时，套接字就会产生 AE_READABLE 事件，引发命令请求处理器执行相应处理。在客户端连接服务器的整个过程中，服务器会一直将客户端套接字的 AE_READABLE 事件与命令请求处理器相关联。

* 命令回复处理器负责将服务器执行命令后得到的回复通过套接字返回给客户端。当服务器由命令回复需要传送给客户端时，服务器会将客户端套接字的 AE_WRITEABLE 事件与命令回复处理器相关联，在客户端准备好接受命令回复时，就会产生 AE_WRITEABLE 事件，引发命令回复处理执行相应处理。在回复命令发送完毕后，服务器就会接触命令回复处理器与客户端套接字的 AE_WRITEABLE事件的关联。

### 时间事件

Redis 的时间事件主要分为两类：

* 定时事件：让程序在指定的时间到达后运行一次

* 周期性事件：让程序每隔一定的时间就运行一次

#### 时间事件的结构

Redis 将所有时间事件都放在一个无序链表中，每次 redis 会遍历整个链表，查找所有已经到达的时间事件，并且调用相应的事件处理器。下面是时间事件的数据结构：

```
typedef struct aeTimeEvent {
	//全局唯一ID
	long long id ;
	// 秒精确的UNIX时间戳，记录时间事件到达的时间
	long when_sec ;
	// 毫秒精确的UNIX时间戳，记录时间事件到达的时间
	long when_ms ;
	// 时间处理器。 如果返回值为AE_NOMORE，事件为定时事件，如果返回值为非AE_NOMORE的整数，则事件为周期性事件
	aeTimeProc *timeProc ;
	// 事件结束回调函数，析构一些资源
	aeEventFinalizerProc *finalizerProc ;
	// 私有数据
	void *clientData ;
	// 前驱节点
	struct aeTimeEvent *prev ;
	// 后继节点
	struct aeTimeEvent *next ;
} aeTimeEvent ;
```

目前 Redis 服务器在一般情况下只执行 serverCron 函数这一个时间事件，并且这个事件是周期性的。serverCron 函数的主要工作包括：

* 更新服务器的各类统计信息，如时间、内存占用、数据库占用等情况

* 清理数据库中的过期键值对

* 关闭和清理连接失效的客户端

* 尝试进行AOF或RDB持久化操作

* 如果服务器是主服务器，那么对从服务器进行定期同步

* 如果处于集群模式，对集群进行定期同步和连接测试

在2.6版本，serverCron 函数平均每隔 100ms 运行一次。

### 事件的调度与执行

文件事件与时间事件之间是合作关系，服务器会轮流处理这两种事件，并且处理事件的过程也不会进行抢占。下面是事件调度的伪代码：

```
def aeProcessEvents():
	//获取达到时间离当前时间最接近的时间事件
	time_event = aeSearchNearestTimer()
	//计算最接近的时间事件距离到达还有多少毫秒
	remaind_ms = time_event.when - unix_ts_now()
	//如果事件已经到达，remaind_ms可能为负数，将它设置为0
	if remaind_ms < 0:
		remaind_ms = 0
	//根据remaind_ms创建timeval结构
	timeval = create_timeval_with_ms(remaind_ms)
	//阻塞并等待文件事件的产生，最大阻塞时间由传入的timeval结构决定
	//如果remaind_ms为0，那么aeApiPoll调用后立马返回，不阻塞
	aeApiPoll(timeval)
	//处理所有已产生的文件事件
	processFileEvents()
	//处理所有已到达的时间事件，因为事件事件是在文件事件完成之后执行
	//所以时间事件的实际处理事件，通常会比设定时间稍晚一些
	processTimeEvents()
```

将上述代码放入一个循环中，再加上初始化和清理函数，这就构成了redis服务器的主函数：

```
def main():
	//初始化服务器
	init_server()
	//一直处理事件，直到服务器关闭
	while server_is_not_shutdown():
		aeProcessEvents()
	//服务器关闭，执行清理操作
	clean_server()
```