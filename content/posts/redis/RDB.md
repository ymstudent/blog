---
title: "Redis RDB持久化"
date: 2021-01-03T20:42:56+08:00
draft: false
tags: ["redis"]
keywords: ["redis", "RDB持久化"]
---

Redis RDB 是把内存中的数据集以快照形式写入磁盘，采用二进制压缩存储。

RDB 把整个 Redis 的数据保存在单一文件中，**比较适合用来做冷备，但缺点是快照保存完成之前如果宕机，这段时间的数据将会丢失，另外保存快照时可能导致服务短时间不可用**。

### RDB文件的创建

* SAVE 命令创建 RDB 文件会导致服务器阻塞，直到文件创建完毕为止，在服务器进程阻塞期间，服务器不能处理任何命令请求。

* BGSVAE 命令会 fork 出一个子进程，然后由子进程负载创建 RDB 文件，服务器进程（父进程）继续处理命令请求。在 BGSAVE 执行期间，客户端发送的 SAVE 和 BGSAVE 命令会被服务器拒绝，以防止产生竞争条件。BGREWRITEAOF 命令会在 BGSAVE 命令执行完毕之后执行。

### RDB文件的载入

redis在启动时如果检测到 RDB 文件的存在，就会自动载入 RDB 文件；但如果服务器开启了 AOF 持久化功能，那么服务器会优先使用AOF 文件来还原数据库。

在 RDB 文件载入期间，服务器会一直处于阻塞状态，直到载入工作完成为止。

### 自动间隔性保存

用户可以通过配置文件设置多个保存条件，只要其中任意一个条件满足，服务器就会执行 BGSAVE 命令。

```
save 900 1 //服务器在900s内，对数据库至少修改了一次
save 300 10 //服务器在300s内，对数据库至少修改了10次
save 60 10000 //服务器在60s内，对数据库至少修改了10000次
```

redis自动间隔保存的实现

```
struct redisServer {
	//记录了保存条件的数组
	struct saveparam *saveparams ;
	//修改计数器，服务器每修改一个键后，都会将dity计数器+1
	long long dity ;
	//上次执行保存的时间
	time_t lastsave ;
}

struct saveparam {
	//秒数
	time_t seconds ;
	//修改数
	int changes ;
}

//自动保存实现伪代码
def serverCron {
	//遍历所有保存条件
	for saveparam in server.saveparams :
	//计算距离上次执行保存操作有多少秒
	save_interval = unixtime_now ( ) - server.lastsave ;
	//如果修改次数和保存间隔都满足，就执行保存操作
	if server.dity >= saveparam.changes and save_interval > saveparam.seconds :
	BGASEVE()
}
```

### RDB文件的结构（了解）

RDB是一个被压缩的二进制文件，它由如下5个部分组成：| REDIS | db_version | database | EOF | check_sum |

| 属性       | 长度            | 作用                                     |
| ---------- | --------------- | ---------------------------------------- |
| REDIS      | 5字节           | RDB文件标识                              |
| db_version | 4字节           | RDB文件版本号                            |
| database   | 不确定          | 包含零个或任意多个数据库以及其中的键值对 |
| EOF        | 1字节           | 文件正式内容结束标识符                   |
| check_sum  | 8字节无符号整数 | 校验和，检查文件是否有出错或损坏的情况   |